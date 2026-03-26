# 腳本、掛勾與進階功能

表格定義除了欄位結構外，還支援多種程式化擴展機制。

## MyTablePolicy 政策驗證

政策是在 insert 和 update 操作前自動觸發的驗證邏輯。

```typescript
{
  name: string;    // 政策名稱
  script: string;  // 指令碼（編譯後的 JavaScript）
  memo?: string;   // 備註
}
```

政策腳本應回傳 `null`（通過）或錯誤訊息字串（失敗）。

## MyTableHook 生命週期掛勾

掛勾在特定操作的前後自動觸發。

### HookType 掛勾類型

| 類型 | 觸發時機 | 說明 |
|------|---------|------|
| `BEFORE_INSERT` | 新增前 | 第一次新增時呼叫，可修改紀錄內容 |
| `BEFORE_UPDATE` | 更新前 | 每次更新前呼叫 |
| `AFTER_INSERT` | 新增後 | 新增完成後呼叫 |
| `AFTER_UPDATE` | 更新後 | 更新完成後呼叫 |
| `MAPPING` | 每次新增或更新 | （已棄用） |

```typescript
{
  type: HookType;       // 掛勾類型
  name: string;         // 掛勾名稱
  lineName?: string;    // 表身名稱（不指定則為表頭）
  script: string;       // 指令碼
  memo?: string;        // 備註
}
```

## MyTableFunction 業務函數

可由使用者或系統觸發的自訂函數。

```typescript
{
  name: string;              // 函數名稱
  script: string;            // 指令碼
  type?: FunctionType;       // 函數類型
  isPublic?: boolean;        // 前端是否可見
  isBulk?: boolean;          // 是否支援右鍵批次執行
  isBatchable?: boolean;     // 是否支援批次作業下拉選單執行
  ignoreApproval?: boolean;  // 執行前是否忽略簽核狀態
  memo?: string;             // 備註
  group?: string;            // 函式群組
  onJobRetry?: number;       // 待辦工作重試次數（預設 0，即第一次錯誤就移除）
}
```

### jobQueues 格式與 queueNo 的關係

`jobQueues` 定義在表格的 `body` 中，格式為逗號分隔的優先權數值字串：

```typescript
body: {
  // 5 個佇列，每個優先權為 1（低優先，適合背景非急迫任務）
  jobQueues: '1,1,1,1,1',

  // 20 個佇列，每個優先權為 10（高優先，適合結帳後需盡快執行的任務）
  jobQueues: Array.from({ length: 20 }, () => '10').join(','),
}
```

- **佇列數量**：字串中逗號分隔的數值個數，決定 `queueNo` 的有效範圍（1 ~ N）
- **優先權值**：每個數值代表該佇列的加權隨機優先權，值越高被 Worker 優先處理的機率越大

建立 TodoJob 時，`queueNo` 必須在 1 ~ N 的範圍內。建議使用 `Utils.generateTodoJobQueueNo` 並傳入 `queryCount = N`（佇列總數）：

```typescript
// 表格定義：10 個佇列（最大上限）
jobQueues: Array.from({ length: 10 }, () => '10').join(','),

// 建立 job 時：queryCount 需對應佇列總數
queueNo: Utils.generateTodoJobQueueNo({ targetName: someKey, queryCount: 10 }),
```

> **重要限制**：`jobQueues` 的佇列數量（陣列長度）**最大為 10**，不可超過。

### functionName 命名規範

- 表格定義中的 `functions[].name`（即 `functionName`）：**使用繁體中文**
- 腳本檔案名稱（`.ts` 檔名）：**使用英文**

```typescript
// 表格定義
functions: [
  {
    name: '更新促銷消費紀錄',         // ✅ 繁體中文
    script: readCompiledScript(resolve(__dirname, './scripts/table/updatePromotionStatistics.ts')),
  },
]

// TodoJob 建立時
functionName: '更新促銷消費紀錄',     // ✅ 需與 functions[].name 完全一致
```
```

### MyTableFunctionArgument 函數參數

函數可以定義輸入參數，結構類似 FieldSchema 但有限制：

- `fieldType` 只能是 `DATA` 或 `REF_KEY`
- `readWrite` 固定為 `INSERT`
- 必須指定 `functionName` 和 `lineName`（`null` 代表表頭）
- 不支援自動編碼相關屬性

## MyTableCron 排程任務

定時執行的腳本。

```typescript
{
  name: string;              // 排程名稱（不含空白）
  script: string;            // 指令碼
  functionName?: string;     // 關聯的函數名稱
  rate?: CronRate;           // 執行頻率
  atUtcTime?: string;        // 指定 UTC 時間（僅 DAILY 時生效）
  memo?: string;             // 備註
  group?: string;            // 排程群組
}
```

## MyTableApprovalChain 簽核路徑

定義簽核流程的各階段和角色。

```typescript
{
  name: string;        // 規則編號
  ruleName: string;    // 簽核鏈名稱
  stage: number;       // 簽核階段
  role: string;        // 簽核角色
  script?: string;     // 腳本
  memo?: string;       // 備註
  reminder?: string;   // 簽核提醒
}
```

## MyTableLib 共用函式庫

跨腳本共用的函式庫。

```typescript
{
  name: string;    // 命名空間，在 script 中以此名稱存取
  script: string;  // 腳本
  memo?: string;   // 備註
}
```

## MyTableEvent 事件

```typescript
{
  name: string;    // 事件名稱
  script: string;  // 指令碼
  memo?: string;   // 備註
}
```

## MyTableFieldGroup 欄位群組

用於在 UI 中將欄位分組顯示。

```typescript
{
  lineName?: string;   // 所屬表身（null 代表表頭）
  name: string;        // 群組名稱
  ordering?: number;   // 排序
  memo?: string;       // 備註
}
```

## MyTableSearchSuggestions 查詢提示

```typescript
{
  name?: string;              // 編號
  displayName?: string;       // 顯示名稱
  searchArgument?: string;    // 預設查詢參數
}
```

## NorwegianForest 腳本檔案結構

在 NorwegianForest 專案中，腳本以獨立的 TypeScript 檔案存放：

```
tables/[模組]/scripts/[表格名稱]/
├── policy.ts           # 政策驗證
├── batchPolicy.ts      # 批次政策驗證
├── beforeInsert.ts     # 新增前掛勾
├── batchBeforeInsert.ts # 批次建立前掛勾
├── beforeUpdate.ts     # 更新前掛勾
├── batchBeforeUpdate.ts # 批次更新前掛勾
├── [function].ts       # 業務函數
└── [function].cron.ts  # 排程任務
```

使用 `readCompiledScript`（來自 `@norwegianForestLibs/compile`）讀取並編譯 TypeScript 腳本：

```typescript
import { readCompiledScript } from '@norwegianForestLibs/compile';
const scriptDir = path.resolve(__dirname, './scripts/myTable');

// 在表格定義中
policies: [
  {
    name: 'validateOrder',
    script: readCompiledScript(path.resolve(scriptDir, 'policy.ts')),
  },
],
```
