# 撰寫 TodoJob 函數腳本

## targetName 的兩種運作模式

`lines.items` 有兩種使用方式，決定了待辦工作的運作模式：

### 模式一：無目標記錄（空陣列）

傳入空陣列 `items: []` 時，工作不會綁定特定記錄。函數收到的 `record` 為空殼物件。此模式通常用於**需要處理複數個記錄的批次工作**，由函數自行查詢與處理目標。

```typescript
// 建立無目標的批次工作
await ctx.query.todoJobsV2({
  data: [{
    body: { tableName: 'pospromotion', functionName: '批次更新統計', ... },
    lines: { items: [] },  // 空陣列：無指定目標
  }],
});

// 函數內自行查詢要處理的記錄
const main: FunctionScript<...> = async (_record, _args, ctx) => {
  const targets = await ctx.query.findV2<...>({ table: 'pospromotion', filters: [...] });
  // 批次處理多筆記錄...
};
```

### 模式二：指定目標記錄（單一 targetName）

傳入帶有 `targetName` 的項目時，系統在執行時會用此值查詢對應表格的記錄，並作為 `record` 傳入函數。

> **重要限制**：`lines.items` 只能包含**最多一個** `targetName`，不可傳入兩個以上。

```typescript
// 建立指定目標的工作
await ctx.query.todoJobsV2({
  data: [{
    body: { tableName: 'pospromotion', functionName: '更新已兌換數量', ... },
    lines: { items: [{ targetName: promotionName }] },  // 僅限一個 targetName
  }],
});

// 函數收到的 record = 查詢到的 pospromotion 記錄
const main: FunctionScript<SafeRecord<PosPromotionBody>> = async (record, args, ctx) => {
  record.body.name  // = promotionName
};
```

**當 targetName 查不到記錄時**，`record` 不會拋錯，而是傳入一個空殼物件（`record.body` 為空）。函數必須自行處理這個情況。

### 模式選擇指引

| 情境 | 模式 | `lines.items` |
|------|------|---------------|
| 針對單一記錄執行操作 | 指定目標 | `[{ targetName: 'xxx' }]` |
| 批次處理多筆記錄 | 無目標 | `[]` |
| 目標記錄可能不存在（upsert） | 指定目標（用觸發來源當 targetName 追溯） | `[{ targetName: saleName }]` |

---

## Upsert 模式：目標記錄可能不存在

當 TodoJob 需要建立或更新一筆統計記錄（如首次消費就建立），不應用 `targetName` 指向那筆記錄（因為它可能還不存在）。正確做法：

1. **`targetName` 使用觸發來源**（例如 `saleName`）提供可追溯性，不依賴它載入記錄
2. **用 `param` 傳遞運行時參數**（promotionName、memberName 等）
3. **在函數內用 `findV2 + session:true` 查詢目標**，再決定 insert 或 update

```typescript
// finishCheckout.ts：建立 job 時用 saleName 當 targetName
todoJobs.push({
  body: {
    tableName: 'pospromotionmemberstatistics',
    functionName: 'updatePromotionStatistics',
    param: JSON.stringify({ promotionName, memberName, saleName }),
    queueNo: Utils.generateTodoJobQueueNo({ targetName: promotionName + memberName, queryCount: 20 }),
  },
  lines: { items: [{ targetName: saleName }] },  // 追溯用，不用來載入記錄
});

// updatePromotionStatistics.ts：函數內自行查詢
const main: FunctionScript<...> = async (_record, _args, ctx) => {
  const { promotionName, memberName, saleName } = JSON.parse(ctx.todoJob?.body.param || '{}');

  const [existing] = await ctx.query.findV2<...>({
    table: 'pospromotionmemberstatistics',
    filters: [{ body: { promotionName: { $in: [promotionName] }, memberName: { $in: [memberName] } } }],
    session: true,  // 在同一交易內讀取，確保冪等性
    limit: 1,
  });

  if (existing) {
    if (existing.body.saleName === saleName) {
      return { status: 'CANCELLED', result: '已計算，跳過' };  // 冪等保護
    }
    await ctx.query.updateV2<...>({ ... });
    return { status: 'DONE', result: '...' };
  } else {
    await ctx.query.insertV2<...>({ ... });
    return { status: 'DONE', result: '...' };
  }
};
```

---

## ctx.todoJob：讀取工作自身資訊

函數內可透過 `ctx.todoJob` 存取當前工作的完整資訊：

| 欄位 | 說明 |
|------|------|
| `ctx.todoJob.body.param` | 建立 job 時傳入的自訂參數（JSON string） |
| `ctx.todoJob.body.buffer` | 前一階段傳遞的暫存資料（WORKING 多階段用） |
| `ctx.todoJob.body.execCount` | 累計執行次數 |

**傳參範例**：
```typescript
// 建立時
param: JSON.stringify({ promotionName, memberName, saleName })

// 函數內讀取
const { promotionName, memberName, saleName } = JSON.parse(ctx.todoJob?.body.param || '{}');
```

---

## session:true 的冪等性保護

在 TodoJob 函數中進行「先查再寫」的操作時，`findV2` 加上 `session: true` 可確保讀取在同一個 MongoDB 交易內，避免在高並發時讀到 commit 前的舊資料：

```typescript
// ✅ 使用 session:true 在交易內讀取，確保冪等性
const [existing] = await ctx.query.findV2<...>({
  table: 'pospromotionmemberstatistics',
  filters: [...],
  session: true,   // 同交易內讀取
  limit: 1,
});

// ❌ 不加 session:true 可能讀到舊資料（另一個並行 job 尚未 commit 的狀態）
```

---

## queueNo 分配策略：防止 race condition

若多個 job 可能對同一筆記錄進行 upsert，需確保它們落在同一個 queue（序列執行）：

```typescript
// ✅ 用業務唯一鍵當 targetName，同一對 (promotion, member) 永遠進同一 queue
queueNo: Utils.generateTodoJobQueueNo({
  targetName: promotionName + memberName,  // 業務唯一鍵
  queryCount: 20,  // 對應表格 jobQueues 的長度
})
```

`generateTodoJobQueueNo` 對 `targetName` 做 hash 後取模，相同的 `targetName` 永遠回傳相同的 `queueNo`。搭配佇列的樂觀鎖機制，同一 queue 同時只有一個 Worker 執行，有效避免 race condition。

> **注意**：`queryCount` 必須等於表格 `jobQueues` 的佇列數量（陣列長度）。若不一致，`queueNo` 可能超出範圍。
