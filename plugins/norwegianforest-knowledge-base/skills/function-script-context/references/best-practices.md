# 撰寫腳本的最佳實踐

## 防循環設計原則

### 1. 追蹤完整觸發鏈再動筆

每次 `batchUpdateV2` 都可能觸發目標表的 `batchBeforeUpdate`（即使 `ignorePolicy:true`）。
在撰寫跨表更新前，先畫出完整的觸發鏈：

```
我的操作
  → 目標表 batchBeforeUpdate（一定執行！）
  → [若 ignorePolicy:false] 目標表 batchPolicy
  → [若 ignorePolicy:false] 目標表 policy（每筆）
```

### 2. 用 sentinel 欄位觸發政策，不要在 hook 中直接執行昂貴邏輯

```typescript
// ❌ 在 hook 中直接執行昂貴的政策計算
// 若此 hook 被觸發多次，昂貴計算也執行多次

// ✅ 更新一個 sentinel 欄位，讓政策自行決定是否執行
await ctx.query.batchUpdateV2<DSVOrderBody>({
  table: 'dsvorder',
  updates: dsvOrderNames.map(name => ({
    key: { name: 'name', value: name },
    body: { updatedAtByTransferOrder: new Date() },  // sentinel
  })),
});

// 在 dsvorder.policy 中：
if (record.body.updatedAtByTransferOrder) {
  // 執行昂貴的庫存計算...
}
```

### 3. INSERT 情境的 oldRecords 判斷

```typescript
// ❌ 錯誤：INSERT 情境中所有 records 都會被視為「有變動」
const changed = records.filter(r => {
  const old = oldRecords?.find(o => o.body.name === r?.body.name);
  return !!(r && (!old || old.body.quantity !== r?.body.quantity));
});
if (changed.length) await ctx.query.batchUpdateV2(...);

// ✅ 正確
if (changed.length && oldRecords) await ctx.query.batchUpdateV2(...);
```

### 4. 旗標命名應反映來源

```typescript
// ❌ 不清楚的旗標名稱
skipDsvorderUpdate: true

// ✅ 清楚反映來源的旗標名稱
isFromDSVOrderDetailBatchPolicy: true
```

---

## 常見錯誤模式對照

### ① ignorePolicy:true 也無法跳過 before hook（beforeInsert / beforeUpdate / batchBeforeInsert / batchBeforeUpdate）

```typescript
// ❌ 錯誤：以為 ignorePolicy:true 可以讓 transferorder 的 batchBeforeUpdate 不執行
await ctx.query.batchUpdateV2({
  table: 'transferorder',
  ignorePolicy: true,  // 只跳過 batchPolicy 和 policy
  updates: [...],
  // 所有 before hook 仍然執行！batchBeforeUpdate 可能觸發跨表更新！
});

// ✅ 正確：用 userContent 旗標讓 batchBeforeUpdate 自行跳過特定邏輯
await ctx.query.batchUpdateV2({
  table: 'transferorder',
  ignorePolicy: true,
  updates: [...],
  user: JSON.stringify({
    ...userContent,
    isFromMyHook: true,
  }),
});
// 並在 transferorder/batchBeforeUpdate.ts 中：
// if (userContent.isFromMyHook) { /* 跳過循環邏輯 */ }
```

### ② batchPolicy 結束後才執行的更新被重複觸發

```typescript
// 情境：batchPolicy 在結尾更新 dsvorder，但 dsvorder.policy 其實在整條鏈結束後才需要執行一次

// ❌ 錯誤：在 batchPolicy 結尾不必要地更新 dsvorder
// 此更新觸發 dsvorder.policy，但後面還有其他更新會再次觸發 dsvorder.policy

// ✅ 正確：確保 dsvorder 的觸發點只有一個（通常在最後的 sentinel 更新）
```

### ③ 沒有把握旗標是否會被 functions 過濾掉

```typescript
// 注意：某些腳本在傳遞 user 時會過濾 functions
user: JSON.stringify({
  ...userContent,
  functions: userContent.functions?.filter(fn => fn !== '總倉自動調撥'),
  // 若上游設定了某個依賴 functions 判斷的邏輯，這裡可能會清掉它
});

// 確保你的自訂旗標不依賴 functions 陣列，而是用獨立的布林旗標
isFromDSVOrderDetailBatchPolicy: true,  // ✅ 獨立旗標，不受 functions 過濾影響
```

---

---

## TodoJob 函數特定建議

### ① 從 finishCheckout 等高頻函數觸發的統計更新，務必用 TodoJob

直接在 `finishCheckout` 中執行統計寫入，若寫入失敗會導致整個結帳失敗。非核心的統計累加應透過 `todoJobsV2` 非同步執行：

```typescript
// ❌ 直接在結帳流程中更新統計
await ctx.query.updateV2<...>({ table: 'statistics', ... });

// ✅ 用 TodoJob 非同步更新，避免失敗影響結帳
todoJobs.push({
  body: { tableName: 'statistics', functionName: 'updateStats', ... },
  lines: { items: [{ targetName: saleName }] },
});
```

### ② TodoJob 函數內的「先查再寫」必須加 session:true

在 TodoJob 函數中查詢後再決定 insert 或 update，`findV2` 需加 `session: true` 才能在同一交易內讀取，防止並行 job 造成重複寫入：

```typescript
const [existing] = await ctx.query.findV2<...>({
  table: '...',
  filters: [...],
  session: true,  // ✅ 必要：在同一交易內讀取
  limit: 1,
});
```

### ③ queueNo 需與 jobQueues 長度一致

`Utils.generateTodoJobQueueNo` 的 `queryCount` 必須等於目標表格 `jobQueues` 的佇列數量，否則會產生超出範圍的 `queueNo`：

```typescript
// 表格定義 20 個佇列 → queryCount = 20
queueNo: Utils.generateTodoJobQueueNo({ targetName: someKey, queryCount: 20 }),
```

---

## 效能考量速查

| 操作 | 約需 DB 查詢數 | 備注 |
|-----|-------------|-----|
| `dsvorder.policy`（完整路徑） | 6~8 次 | 含 getDSVInventoryAmounts、getDSVInTransitInfo 等並行查詢 |
| `dsvorderdetail.batchPolicy`（有 needFill） | 4~5 次 + N次循環 | 每個批號分配可能需要多次查詢 |
| `transferorder.policy`（upsertDSVOrder 路徑） | 4~6 次 | 含 dsvorder 查詢、item 查詢等 |
| 整條 WH push lifecycle（修正後） | ~20-25 次 | 含所有嵌套調用 |

**逾時警戒線**：~50 次 DB 查詢 ≈ 40~60 秒，逼近 120 秒限制

建議：若單次操作的查詢數超過 30 次，就需要評估是否有可消除的冗餘觸發。
