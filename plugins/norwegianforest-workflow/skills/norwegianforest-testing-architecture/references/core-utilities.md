# 核心工具函式

所有工具函式定義在 `tests/utils.ts`。

## generateExecuteFunc\<Args\>(config)

**最重要的工具函式**。讀取編譯後的腳本、進行覆蓋率儀表化，回傳一個可直接呼叫的非同步執行函式。

### 參數

```typescript
function generateExecuteFunc<Args extends MySandboxExecute['arguments']>(configs: {
  /** 腳本原始碼路徑（.ts，會自動找到對應的 webpack 編譯產物） */
  scriptPath: string;
  /** 共用函式庫（對應表格定義中的 libs 設定） */
  libs?: { name: string; path: string }[];
  /** 不計算測試覆蓋率（預設 false） */
  skipCoverage?: boolean;
  /** 在腳本前面注入的程式碼（例如注入 config 變數） */
  scriptPrefix?: string;
})
```

### 回傳的執行函式

```typescript
(options: {
  args: Args;                    // 傳入腳本的參數（必填）
  user?: UserRecord;             // 模擬使用者
  todoJob?: TodoJobRecord;       // TodoJob 記錄（用於 TodoJob 函式）
  param?: Record<string, any>;   // 額外參數
  env?: Record<string, string>;  // 環境變數
  scriptType?: string;           // 腳本類型
}) => Promise<unknown>
```

### 使用範例

```typescript
// 定義執行函式（通常放在 describe 區塊的最上方）
const executeFunc = generateExecuteFunc<[SafeRecord<PosSaleBody, PosSaleLines>]>({
  scriptPath: resolve(__dirname, '../../tables/pos/scripts/possale/policy.ts'),
  libs: [
    { name: 'dayjs', path: resolve(__dirname, '../../tables/common/dayjs.ts') },
    { name: 'Pos', path: resolve(__dirname, '../../tables/common/pos.ts') },
  ],
});

// 在測試中呼叫
const result = await executeFunc({ args: [testSale] });
```

### 內部流程

1. 透過 `readCompiledScript()` 讀取 webpack 編譯後的 JS
2. 用 `nyc instrument` 儀表化（除非 `skipCoverage: true`）
3. 如果有 `scriptPrefix`，將其前置到腳本程式碼
4. 呼叫 `MySandbox.execute()` 在 VM 沙盒中執行
5. 透過 `pick(global, '__coverage__')` 將覆蓋率資料傳入沙盒
6. 如果執行拋出例外，捕捉並回傳錯誤訊息字串

### 關於 libs

`libs` 必須與表格定義中的設定一致。查看方式：

1. 找到表格定義檔案（例如 `tables/pos/possale.ts`）
2. 查看其中的 `scripts` 設定，找到對應腳本的 `libs` 欄位
3. 將列出的 lib name 與 path 對應起來

常見的 libs：

| name | path | 用途 |
|------|------|------|
| `dayjs` | `tables/common/dayjs.ts` | 日期處理 |
| `Utils` | `tables/common/utils.ts` | 通用工具函式 |
| `Validator` | `tables/common/validator.ts` | 驗證工具 |
| `Pos` | `tables/common/pos.ts` | POS 相關通用函式 |
| `Procurement` | `tables/common/procurement.ts` | 採購相關通用函式 |

---

## executeFuncUntilDoneOrError(params)

專門用於 **TodoJob 函式**的測試。TodoJob 可能需要多次迭代（status 為 `WORKING` 時持續執行），此函式會自動迴圈直到狀態為 `DONE`、`CANCELLED` 或 `ERROR`。

### 參數

```typescript
function executeFuncUntilDoneOrError<Args extends unknown[]>(params: {
  executeFunc: ReturnType<typeof generateExecuteFunc>;  // generateExecuteFunc 的回傳值
  args: Args;                                           // 傳入腳本的參數
  todoJob: TodoJobRecord;                               // TodoJob 記錄（buffer 會自動傳遞）
}): Promise<{ status: 'DONE' | 'CANCELLED' | 'ERROR'; buffer?: any }>
```

### 使用範例

```typescript
const mockTodoJob = cloneDeep(baseMockTodoJob);
const result = await executeFuncUntilDoneOrError({
  executeFunc,
  args: [testRecord, null],
  todoJob: mockTodoJob,
});
expect(result.status).to.equal('DONE');
```

### 內部行為

1. 將 `todoJob.body.status` 設為 `'WORKING'`
2. 迴圈呼叫 `executeFunc`，每次傳入 `todoJob`
3. 如果回傳的 `buffer` 存在，寫入 `todoJob.body.buffer`（供下次迭代使用）
4. 當 status 不再是 `'WORKING'` 時停止
5. 斷言最終 status 為 `'DONE'`、`'CANCELLED'` 或 `'ERROR'`

---

## findApprovalRule(record, tableName, folder)

用於測試 **approval.ts** 簽核鏈腳本。直接讀取編譯後的簽核腳本並在 Sandbox 中執行，不需要自行設定 `generateExecuteFunc`。

### 參數

```typescript
function findApprovalRule(
  record: SafeRecord | SafeRecord2,     // 要判斷的記錄
  tableName: ASSIGNED_TABLE_NAME,       // 表格名稱常數（如 TABLE_NAME.REQUISITION）
  folder: string                         // tables/ 下的資料夾名稱（如 'procurement'）
): Promise<string>                       // 回傳匹配的簽核鏈名稱
```

### 使用範例

```typescript
import { findApprovalRule } from '../utils';
import { TABLE_NAME } from '@norwegianForestTables/const';

const ruleName = await findApprovalRule(
  mockRecord,
  TABLE_NAME.REQUISITION,
  'procurement'
);
expect(ruleName).to.equal('萬達國際部');
```

### 內部行為

會從 `tables/${folder}/scripts/${tableName}/approval.ts` 讀取編譯後腳本，透過 `MySandbox.execute` 執行。

---

## writeRecord\<R\>(source, query)

將 `UpdateQuery` / `UpdateV2Query` 的變更寫入 mock 資料。常搭配 `MySandbox.queryProcessor` 使用。

### 參數

```typescript
function writeRecord<R extends SafeRecord | SafeRecord2>(
  source: R,                                                    // 目標記錄（會被直接修改）
  query: Omit<UpdateQuery, 'table' | 'method'>                 // 更新查詢
       | Omit<UpdateV2Query, 'table' | 'method'>
): R
```

### 支援的操作

| 操作 | 說明 |
|------|------|
| `query.body` | `Object.assign` 到 `source.body` |
| `query.lines[lineName].$push` | 新增表身行（自動分配 `_id`） |
| `query.lines[lineName].$set` | 修改表身行（依 `key.name` + `key.value` 定位） |
| `query.lines[lineName].$pull` | 刪除表身行（依 `key.name` + `key.value` 定位） |

### 使用範例

```typescript
MySandbox.queryProcessor = async (query) => {
  if (query.method === 'updateV2' && query.table === 'possale') {
    writeRecord(testSale, query);   // 將更新寫入 testSale
    return [1];                     // 回傳影響筆數
  }
};
```

---

## sortRecords\<R\>(records, sortings)

根據 `FindV2Query` 的 `multiSort` 設定排序記錄。回傳排序後的副本（不修改原陣列）。

```typescript
function sortRecords<R extends SafeRecord2>(
  records: R[],
  sortings: FindV2Query['multiSort']   // [{ field: string, direction: 1 | -1 }]
): R[]
```

---

## QueryEventEmitter

帶型別的 EventEmitter，用於追蹤 queryProcessor 中觸發的查詢。詳見 [query-processor-mock.md](query-processor-mock.md#queryeventemitter-事件追蹤)。
