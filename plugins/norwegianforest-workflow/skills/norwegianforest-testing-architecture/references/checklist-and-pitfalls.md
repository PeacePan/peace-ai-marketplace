# 撰寫測試步驟清單與常見陷阱

## 撰寫測試的步驟清單

### 步驟 1：確認腳本類型

| 腳本類型 | 測試檔名 | 需要的工具 |
|---------|---------|-----------|
| policy | `policy.test.ts` | `generateExecuteFunc` + `queryProcessor` |
| beforeInsert | `beforeInsert.test.ts` | `generateExecuteFunc` + `queryProcessor` |
| beforeUpdate | `beforeUpdate.test.ts` | `generateExecuteFunc` + `queryProcessor` |
| function | `functions/<name>.test.ts` | `generateExecuteFunc` + `queryProcessor` |
| approval | `approval.test.ts` | `findApprovalRule`（內建） |
| batchPolicy | `batchPolicy.test.ts` | `generateExecuteFunc` + `queryProcessor` |
| cron（純函式） | `<name>.cron.test.ts` | 直接 import |
| todoJob | `functions/<name>.test.ts` | `generateExecuteFunc` + `executeFuncUntilDoneOrError` + `queryProcessor` |

### 步驟 2：建立測試檔案

在 `tests/<tableName>/` 下建立對應的測試檔案。如果是 function 類型，放在 `functions/` 子資料夾。

### 步驟 3：建立或擴充 mock 資料

在 `tests/<tableName>/mocks.ts` 定義 mock 記錄。如果已有 `mocks.ts`，在其中新增需要的 mock。

```typescript
export const mockMyRecord: SafeRecord2<MyBody, MyLines> = {
  _id: 'test-001',
  body: { /* 必填欄位 */ },
  lines: { /* 表身資料 */ },
};
```

### 步驟 4：設定 generateExecuteFunc

```typescript
const executeFunc = generateExecuteFunc<[SafeRecord2<MyBody, MyLines>]>({
  scriptPath: resolve(__dirname, '../../tables/<folder>/scripts/<tableName>/<scriptName>.ts'),
  libs: [/* 查看表格定義中的 libs 設定 */],
});
```

**查找 libs 的方式**：
1. 開啟表格定義檔（如 `tables/pos/possale.ts`）
2. 找到 `scripts` 或 `policies` 等區段中對應腳本的 `libs` 設定
3. 將 lib name 對應到 `tables/common/` 下的檔案路徑

### 步驟 5：設定 MySandbox.queryProcessor

在 `before()` 中覆寫，在 `after()` 中還原。處理所有腳本會發出的查詢。

**技巧**：先讓所有未知查詢 throw error，執行測試後根據錯誤訊息補充 mock。

### 步驟 6：在 beforeEach 中重置 mock 資料

```typescript
beforeEach(() => {
  testRecord = cloneDeep(mockRecord);
});
```

### 步驟 7：撰寫測試案例

- 調整 mock 資料 → 執行 → 用 `expect` 斷言結果
- 考慮正向（通過）和反向（失敗）情境
- 考慮邊界值和特殊情境

### 步驟 8：執行測試

```bash
npm test -- tests/<tableName>/<scriptName>.test.ts
```

---

## 常見陷阱與注意事項

### 1. 必須先 build

測試前需要 `npm run build`（webpack 編譯），`generateExecuteFunc` 讀取的是**編譯後的 JS 檔案**。`bin/test.sh` 已自動處理，但如果直接用 `npx mocha` 跑會失敗。

**症狀**：`Error: ENOENT: no such file or directory` 指向 `dist/` 路徑

### 2. queryProcessor 必須還原

```typescript
// 正確做法
const originQueryProcessor = MySandbox.queryProcessor;
before(() => { MySandbox.queryProcessor = myMock; });
after(() => { MySandbox.queryProcessor = originQueryProcessor; });
```

**症狀**：其他測試檔案莫名失敗，或收到「未處理的查詢」錯誤

### 3. cloneDeep 避免測試間汙染

```typescript
// 正確做法
let testData = cloneDeep(mockData);
beforeEach(() => { testData = cloneDeep(mockData); });

// 錯誤做法：直接修改 mock 原始物件
testData = mockData;  // ← 會影響其他測試
```

**症狀**：測試單獨跑過但一起跑會失敗，或測試順序不同結果不同

### 4. 未處理的查詢應拋出錯誤

在 `queryProcessor` 的最後加上：

```typescript
throw new Error('未處理的查詢: ' + JSON.stringify(query));
```

**好處**：快速發現遺漏的 mock，避免腳本收到 `undefined` 導致不明確的錯誤

### 5. call 方法的回傳值需要 JSON.stringify

```typescript
// 正確
if (query.method === 'call') {
  return JSON.stringify({ result: 'ok' });
}

// 錯誤：腳本端會收到 "[object Object]"
if (query.method === 'call') {
  return { result: 'ok' };
}
```

**原因**：`ctx.query.call()` 在 Sandbox 內部會對結果做 `JSON.parse`

### 6. 逾時問題

- 預設逾時 30 秒（`-t 30000`）
- 超過 5 秒視為慢測試（`-s 5000`）

**解法**：簡化 mock 資料（減少迭代次數），而非調高逾時

### 7. 平行執行注意事項

Mocha 使用 `--parallel --jobs 2` 執行。每個**測試檔案**在獨立的 worker 中運行，因此：
- `MySandbox.queryProcessor` 的覆寫**不會**跨檔案影響
- 同一檔案內的測試**仍共用**狀態，需注意 `beforeEach` 重置

### 8. 全域變數缺失

如果測試中出現 `ReferenceError: dayjs is not defined`（不是在 VM 內），檢查 `tests/global-setup.ts` 是否有注入。

### 9. libs 對應不正確

**症狀**：VM 內報 `ReferenceError: Utils is not defined`

**原因**：`generateExecuteFunc` 的 `libs` 沒有包含腳本依賴的共用函式庫

**解法**：檢查表格定義中腳本的 `libs` 設定，確保測試中有完整列出

### 10. BSON 序列化限制

VM 與主程序之間使用 BSON 傳遞資料，注意：
- `undefined` 會被轉為 `null`
- `Date` 物件會被正確序列化
- 函式（function）無法序列化
- 循環引用會導致錯誤
