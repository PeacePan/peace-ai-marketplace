---
name: norwegianforest-testing-architecture
description: >
  NorwegianForest 專案測試架構完整指南。涵蓋 Mocha + Chai 測試框架、Node.js VM Sandbox
  隔離執行機制、generateExecuteFunc 工具函式、QueryProcessor mock 模式、
  QueryEventEmitter 事件追蹤、NYC 覆蓋率儀表化，以及 Policy / Hook / Function / Approval /
  Cron / TodoJob 各類腳本的測試範例與最佳實踐。
  當需要為 NorwegianForest 撰寫新測試、修改既有測試、或理解測試架構時，務必先讀此 skill。
  適用場景包括：撰寫 policy 測試、撰寫 function 測試、撰寫 approval 測試、
  撰寫 cron 測試、撰寫 TodoJob 測試、除錯測試失敗、理解 VM sandbox mock 機制。
---

# NorwegianForest 測試架構

NorwegianForest 的表格腳本（policy、beforeInsert、beforeUpdate、function、approval、cron 等）在正式環境中運行於 JavaCat 的 **VM Sandbox**（基於 vm2 的 NodeVM）內。測試必須透過相同的 Sandbox 機制執行，才能確保測試結果與正式環境一致。

## 目錄

1. [核心概念：為什麼需要 VM](#1-核心概念為什麼需要-vm)
2. [技術堆疊](#2-技術堆疊)
3. [目錄結構](#3-目錄結構)
4. [測試執行流程](#4-測試執行流程)
5. [核心工具函式](#5-核心工具函式)
6. [Mock 模式：QueryProcessor](#6-mock-模式queryprocessor)
7. [事件追蹤：QueryEventEmitter](#7-事件追蹤queryeventemitter)
8. [各類腳本測試範例](#8-各類腳本測試範例)
9. [撰寫測試的步驟清單](#9-撰寫測試的步驟清單)
10. [常見陷阱與注意事項](#10-常見陷阱與注意事項)

---

## 1. 核心概念：為什麼需要 VM

NorwegianForest 的腳本**不是**普通的 Node.js 模組。它們在正式環境中被編譯後載入 vm2 的 `NodeVM` 沙盒執行，沙盒提供：

- **隔離性**：腳本無法存取 Node.js 主程序的全域物件
- **受控的 I/O**：所有資料庫操作透過 `ctx.query` 介面，經 MessagePort 傳回主執行緒處理
- **凍結的共用函式庫**：`dayjs`、`Utils` 等 libs 會被 `vm.freeze()` 注入，腳本無法修改

測試時必須走相同的 Sandbox 路徑（`MySandbox.execute`），**只 mock 資料庫查詢（QueryProcessor）**，讓業務邏輯在真實的沙盒環境中執行。

```
測試程式 ──呼叫──▶ MySandbox.execute ──VM 沙盒執行──▶ 腳本邏輯
                                                          │
                                                    ctx.query.*
                                                          │
                                                    ◀── MessagePort ──▶
                                                          │
                                              MySandbox.queryProcessor（被 mock）
```

---

## 2. 技術堆疊

| 元件 | 版本/工具 | 用途 |
|------|----------|------|
| 測試框架 | **Mocha 10** | 測試執行器，支援平行執行（`--parallel --jobs 2`） |
| 斷言庫 | **Chai 4** | `expect` 風格斷言 |
| 覆蓋率 | **NYC (Istanbul) 15** | 腳本儀表化 + 覆蓋率報告 |
| TypeScript | **ts-node** (transpile-only) | 測試檔案直接使用 TypeScript |
| 沙盒執行 | **vm2 NodeVM** (透過 JavaCat MySandbox) | 腳本隔離執行 |
| 資料複製 | **lodash.cloneDeep** | 每個測試案例的 mock 資料獨立 |

---

## 3. 目錄結構

```
NorwegianForest/
├── bin/test.sh                    # 測試啟動腳本
├── .nycrc                         # NYC 覆蓋率設定
├── tests/
│   ├── global-setup.ts            # 全域變數初始化（dayjs, Utils, Procurement）
│   ├── utils.ts                   # 測試工具函式（核心）
│   ├── example.test.ts            # 範例測試（新手參考）
│   ├── env.testing.norcat.yml     # 測試環境變數
│   │
│   ├── possale/                   # ← 每個表格一個資料夾
│   │   ├── mocks.ts              # mock 資料定義
│   │   ├── policy.test.ts        # policy 腳本測試
│   │   ├── beforeInsert.test.ts  # beforeInsert 腳本測試
│   │   ├── beforeUpdate.test.ts  # beforeUpdate 腳本測試
│   │   └── functions/            # function 腳本測試
│   │       ├── balance.test.ts
│   │       └── finishCheckout.test.ts
│   │
│   ├── requisition/
│   │   ├── approval.test.ts      # 簽核鏈測試
│   │   └── functions/
│   │       └── req2po.test.ts
│   │
│   ├── batchmutation/
│   │   ├── mocks.ts
│   │   └── functions/
│   │       └── process.test.ts   # TodoJob 函式測試
│   │
│   └── vendor/
│       ├── note.cron.test.ts     # cron 腳本測試（直接 import 函式）
│       └── monthlyNote.cron.test.ts
│
├── tables/                        # 被測試的腳本原始碼
│   ├── pos/scripts/possale/
│   │   ├── policy.ts
│   │   ├── beforeInsert.ts
│   │   ├── beforeUpdate.ts
│   │   └── ...
│   └── common/                    # 共用函式庫（會以 libs 注入 VM）
│       ├── dayjs.ts
│       ├── utils.ts
│       └── validator.ts
```

**命名慣例**：
- 測試檔案：`<scriptName>.test.ts`（如 `policy.test.ts`、`beforeInsert.test.ts`）
- Cron 測試：`<name>.cron.test.ts`
- Mock 資料：`mocks.ts`（放在對應表格的測試資料夾中）
- Function 測試：放在 `functions/` 子資料夾中

---

## 4. 測試執行流程

### 4.1 執行命令

```bash
# 執行全部測試
npm test

# 執行特定資料夾
npm test -- tests/possale

# 執行特定檔案
npm test -- tests/possale/policy.test.ts
```

### 4.2 啟動流程（bin/test.sh）

```bash
# 1. 建置 JavaCat worker threads pool
cd ../JavaCat && npm run build:worker

# 2. 設定環境變數
export TZ=Greenwich           # 統一時區
export SANDBOX_QUIET=1        # 關閉 Sandbox 冗餘訊息

# 3. 建置 NorwegianForest（webpack 編譯腳本）
cd ../NorwegianForest && npm run build

# 4. 載入環境設定檔
export MOCHA_ENV_FILES="...env.testing.yml,...env.testing.norcat.yml"

# 5. 執行 Mocha（平行 2 個 worker、30 秒逾時）
npx nyc mocha --parallel --jobs 2 \
  -r ts-node/register/transpile-only \
  -r ../JavaCat/test/envYamlLoader.ts \
  -r ../JavaCat/test/mocha.fixture.ts \
  -r tsconfig-paths/register \
  -r tests/global-setup.ts \
  -s 5000 -t 30000 --exit \
  "${TARGET_DIR}"
```

### 4.3 Mocha 載入順序

1. **ts-node/register/transpile-only** — TypeScript 編譯
2. **envYamlLoader.ts** — 載入 YAML 環境設定
3. **mocha.fixture.ts** — 初始化 `global.testTools`（context、factory 等）
4. **tsconfig-paths/register** — 路徑別名解析
5. **global-setup.ts** — 注入全域變數（`global.dayjs`、`global.Utils`、`global.Procurement`）

---

## 5. 核心工具函式

所有工具函式定義在 `tests/utils.ts`。

### 5.1 `generateExecuteFunc<Args>(config)`

**最重要的工具函式**。讀取編譯後的腳本、進行覆蓋率儀表化，回傳一個可直接呼叫的非同步執行函式。

```typescript
const executeFunc = generateExecuteFunc<[SafeRecord<MyBody, MyLines>]>({
  // 腳本原始碼路徑（.ts，會自動找到對應的 webpack 編譯產物）
  scriptPath: resolve(__dirname, '../../tables/pos/scripts/possale/policy.ts'),
  // 注入的共用函式庫
  libs: [
    { name: 'dayjs', path: resolve(__dirname, '../../tables/common/dayjs.ts') },
    { name: 'Utils', path: resolve(__dirname, '../../tables/common/utils.ts') },
  ],
  // 可選：跳過覆蓋率（預設 false）
  skipCoverage: false,
  // 可選：在腳本前注入程式碼
  scriptPrefix: '',
});

// 使用方式
const result = await executeFunc({
  args: [testRecord],         // 傳入腳本的參數
  user: mockUser,             // 可選：模擬使用者
  todoJob: mockTodoJob,       // 可選：TodoJob 記錄（用於 TodoJob 函式）
  param: { key: 'value' },    // 可選：額外參數
  env: { ENV_VAR: 'val' },    // 可選：環境變數
  scriptType: 'function',     // 可選：腳本類型
});
```

**內部流程**：
1. 透過 `readCompiledScript()` 讀取 webpack 編譯後的 JS
2. 用 `nyc instrument` 儀表化（除非 `skipCoverage: true`）
3. 呼叫 `MySandbox.execute()` 在 VM 沙盒中執行
4. 透過 `pick(global, '__coverage__')` 將覆蓋率資料傳入沙盒
5. 如果執行拋出例外，捕捉並回傳錯誤訊息字串

### 5.2 `executeFuncUntilDoneOrError(params)`

專門用於 **TodoJob 函式**的測試。TodoJob 可能需要多次迭代（status 為 `WORKING` 時持續執行），此函式會自動迴圈直到狀態為 `DONE`、`CANCELLED` 或 `ERROR`。

```typescript
const result = await executeFuncUntilDoneOrError({
  executeFunc,
  args: [testRecord, null],
  todoJob: mockTodoJob,   // 需要提供 TodoJob 記錄，buffer 會自動傳遞
});
expect(result.status).to.equal('DONE');
```

### 5.3 `findApprovalRule(record, tableName, folder)`

用於測試 **approval.ts** 簽核鏈腳本。直接讀取編譯後的簽核腳本並在 Sandbox 中執行。

```typescript
const ruleName = await findApprovalRule(
  mockRecord,
  TABLE_NAME.REQUISITION,   // 表格名稱常數
  'procurement'              // tables/ 下的資料夾名稱
);
expect(ruleName).to.equal('萬達國際部');
```

### 5.4 `writeRecord<R>(source, query)`

將 `UpdateQuery` / `UpdateV2Query` 的變更寫入 mock 資料。常搭配 `MySandbox.queryProcessor` 使用，讓 mock 的 update 操作實際修改測試資料。

```typescript
MySandbox.queryProcessor = async (query) => {
  if (query.method === 'updateV2' && query.table === 'possale') {
    writeRecord(testSale, query);   // 將更新寫入 testSale
    return [1];                     // 回傳影響筆數
  }
};
```

**支援的操作**：
- `query.body` → 直接 `Object.assign` 到 `source.body`
- `query.lines[lineName].$push` → 新增表身行
- `query.lines[lineName].$set` → 修改表身行
- `query.lines[lineName].$pull` → 刪除表身行

### 5.5 `sortRecords<R>(records, sortings)`

根據 `FindV2Query` 的 `multiSort` 設定排序記錄。用於驗證查詢結果排序。

### 5.6 `QueryEventEmitter`

帶型別的 EventEmitter，用於追蹤 queryProcessor 中觸發的查詢。詳見[第 7 節](#7-事件追蹤queryeventemitter)。

---

## 6. Mock 模式：QueryProcessor

### 6.1 基本概念

`MySandbox.queryProcessor` 是一個**全域的非同步回呼函式**，所有在 VM 沙盒中透過 `ctx.query.*` 發出的資料庫操作都會被路由到這裡。測試時覆寫它來攔截並模擬查詢結果。

### 6.2 標準模式

```typescript
import { MySandbox } from '@javaCat/MySandbox';

describe('我的測試', () => {
  // 保留原始 queryProcessor（重要！）
  const originQueryProcessor = MySandbox.queryProcessor;

  before(() => {
    MySandbox.queryProcessor = async (query): Promise<unknown> => {
      // 根據 query.method 和 query.table 回傳對應的 mock 資料
      if (query.method === 'findV2') {
        if (query.table === 'possale') return [testSale];
        if (query.table === 'posstore') return [mockStore];
        return [];  // 預設回傳空陣列
      }
      if (query.method === 'updateV2') {
        if (query.table === 'possale') {
          writeRecord(testSale, query);
          return 1;  // 回傳影響筆數
        }
      }
      if (query.method === 'insertV2') {
        return 'new-id-001';  // 回傳新建記錄 ID
      }
      if (query.method === 'batchInsertV2') {
        return query.inserts.map((_, i) => `batch-id-${i}`);
      }
      if (query.method === 'call') {
        if (query.name === '我的函式') return JSON.stringify({ result: 'ok' });
      }
      // 未處理的查詢拋出錯誤，幫助發現遺漏的 mock
      throw new Error('未處理的查詢: ' + JSON.stringify(query));
    };
  });

  after(() => {
    // 還原（必要！避免影響其他測試）
    MySandbox.queryProcessor = originQueryProcessor;
  });
});
```

### 6.3 各 query method 的回傳值規範

| method | 回傳型別 | 說明 |
|--------|---------|------|
| `findV2` / `find` | `Record[]` | 回傳記錄陣列 |
| `insertV2` / `insert` | `string` | 新建記錄的 ID |
| `batchInsertV2` | `string[]` | 多筆新建記錄的 ID 陣列 |
| `updateV2` / `update` | `number` | 影響的記錄數 |
| `batchUpdateV2` | `number` | 影響的記錄數 |
| `call` | `string` (JSON) | 函式回傳值（需 JSON.stringify） |
| `echo` | `void` | 無需回傳 |
| `archiveV2` / `unarchiveV2` | `number` | 影響的記錄數 |
| `todoJobsV2` | `TodoJob[]` | 待辦工作陣列 |
| `setTodoJob` | `void` | 無需回傳 |

---

## 7. 事件追蹤：QueryEventEmitter

當你需要**驗證腳本是否發出了預期的查詢**時，使用 `QueryEventEmitter`。

```typescript
import { QueryEventEmitter, generateExecuteFunc } from '../utils';

describe('事件追蹤範例', () => {
  const eventEmitter = new QueryEventEmitter();

  before(() => {
    MySandbox.queryProcessor = async (query): Promise<unknown> => {
      // 每個查詢都 emit 事件
      eventEmitter.emit(query.method, query);

      if (query.method === 'findV2') return [];
      if (query.method === 'updateV2') return 1;
      // ...
    };
  });

  it('應該更新寵速配訂單', async () => {
    let hasUpdatedFastBuyOrder = false;

    eventEmitter.addListener('updateV2', (query) => {
      hasUpdatedFastBuyOrder ||=
        query.table === 'mafastbuyorder' &&
        query.body?.saleName === testSale.body.name;
    });

    await executeFunc({ args: [testSale] });

    eventEmitter.removeAllListeners('updateV2');
    expect(hasUpdatedFastBuyOrder).to.be.true;
  });
});
```

**支援的事件名稱**：`echo`、`find`、`findV2`、`insert`、`insertV2`、`batchInsertV2`、`update`、`updateV2`、`batchUpdateV2`、`dangerBatchUpdateV2`、`call`、`archiveV2`、`unarchiveV2`、`todoJobsV2`、`setTodoJob`、`publish`、`publishV2`、`http.json` 等。

---

## 8. 各類腳本測試範例

### 8.1 Policy 腳本測試

Policy 腳本接收一筆記錄，驗證後回傳錯誤訊息（`string`）或 `null`/`undefined`（通過）。

```typescript
import { expect } from 'chai';
import { cloneDeep } from 'lodash';
import { resolve } from 'path';
import { MySandbox } from '@javaCat/MySandbox';
import { generateExecuteFunc } from '../utils';
import { mockStore } from './mocks';

describe('門市政策測試', () => {
  let testStore = cloneDeep(mockStore);
  const originQueryProcessor = MySandbox.queryProcessor;

  const executeFunc = generateExecuteFunc<[SafeRecord2<MyBody, MyLines>]>({
    scriptPath: resolve(__dirname, '../../tables/myFolder/scripts/myTable/policy.ts'),
    libs: [],  // 如果腳本用到 dayjs、Utils 等，需要在此列出
  });

  before(() => {
    MySandbox.queryProcessor = async (query): Promise<unknown> => {
      if (query.method === 'findV2') return [];
      throw new Error('未處理的查詢: ' + JSON.stringify(query));
    };
  });

  beforeEach(() => {
    testStore = cloneDeep(mockStore);  // 每個測試重置 mock 資料
  });

  after(() => {
    MySandbox.queryProcessor = originQueryProcessor;
  });

  it('合法資料應通過政策檢查', async () => {
    const result = await executeFunc({ args: [testStore] });
    expect(result).to.be.null;  // null 或 undefined 代表通過
  });

  it('非法資料應回傳錯誤訊息', async () => {
    testStore.body.someField = 'invalid';
    const result = await executeFunc({ args: [testStore] });
    expect(result).to.include('錯誤訊息關鍵字');
  });
});
```

### 8.2 BeforeInsert / BeforeUpdate 腳本測試

```typescript
const executeFunc = generateExecuteFunc<[SafeRecord2<MyBody>]>({
  scriptPath: resolve(__dirname, '../../tables/myFolder/scripts/myTable/beforeInsert.ts'),
  libs: [{ name: 'dayjs', path: resolve(__dirname, '../../tables/common/dayjs.ts') }],
});

it('新增前應自動填入欄位', async () => {
  const result = await executeFunc({ args: [testRecord] }) as SafeRecord2<MyBody>;
  expect(result.body.autoField).to.equal('expectedValue');
});
```

### 8.3 Function 腳本測試

Function 腳本的參數格式為 `[record, argument]`。

```typescript
import batchProcess from '@norwegianForestTables/function/scripts/batchmutation/process';
type FuncArgs = [Parameters<typeof batchProcess>[0], null];

const executeFunc = generateExecuteFunc<FuncArgs>({
  scriptPath: resolve(tablesRoot, 'function/scripts/batchmutation/process.ts'),
  libs: [
    { name: 'dayjs', path: resolve(tablesRoot, 'common/dayjs.ts') },
    { name: 'Utils', path: resolve(tablesRoot, 'common/utils.ts') },
  ],
});

it('應成功處理批次作業', async () => {
  const result = await executeFunc({ args: [testInput, null] });
  expect(result).to.deep.include({ success: true });
});
```

### 8.4 Approval 腳本測試

使用 `findApprovalRule` 工具函式，不需要自行設定 `generateExecuteFunc`。

```typescript
import { findApprovalRule } from '../utils';
import { TABLE_NAME } from '@norwegianForestTables/const';

it('IT 請購單應走萬達國際部簽核鏈', async () => {
  mockRecord.body.subsidiary = VendorSubsidiary.WP;
  mockRecord.body.type = RequisitionType.IT;
  const ruleName = await findApprovalRule(mockRecord, TABLE_NAME.REQUISITION, 'procurement');
  expect(ruleName).to.equal('萬達國際部');
});
```

### 8.5 Cron 腳本測試

Cron 腳本通常會 export 純函式，可以**直接 import 測試**，不一定需要走 VM Sandbox。

```typescript
import { checkDateIsValidToAddWeekBillNote } from '@norwegianForestTables/procurement/scripts/vendor/note.cron';
import dayjs from '@norwegianForestTables/common/dayjs';

it('週對帳通知單只在週一會觸發', () => {
  const monday = dayjs('2023-01-30T00:00:00+08:00').toDate();
  expect(checkDateIsValidToAddWeekBillNote(monday)).to.be.true;
});

it('週二~週日不會觸發', () => {
  const tuesday = dayjs('2023-01-31T04:00:00+08:00').toDate();
  expect(checkDateIsValidToAddWeekBillNote(tuesday)).to.be.false;
});
```

### 8.6 TodoJob 函式測試

使用 `executeFuncUntilDoneOrError` 處理多階段執行。

```typescript
import { executeFuncUntilDoneOrError, generateExecuteFunc } from 'tests/utils';

const executeFunc = generateExecuteFunc<MyFuncArgs>({
  scriptPath: resolve(tablesRoot, 'function/scripts/myTable/myTodoJob.ts'),
  libs: [...],
});

it('待辦工作應執行完成', async () => {
  const result = await executeFuncUntilDoneOrError({
    executeFunc,
    args: [testInput, null],
    todoJob: cloneDeep(mockTodoJob),  // todoJob.body.buffer 會自動在迭代間傳遞
  });
  expect(result.status).to.equal('DONE');
});
```

### 8.7 BatchPolicy 腳本測試

```typescript
const executeFunc = generateExecuteFunc<[{ records: SafeRecord2[] }]>({
  scriptPath: resolve(__dirname, '../../tables/myFolder/scripts/myTable/batchPolicy.ts'),
  libs: [],
});

it('批次驗證應通過', async () => {
  const result = await executeFunc({ args: [{ records: [testRecord1, testRecord2] }] });
  expect(result).to.be.null;
});
```

---

## 9. 撰寫測試的步驟清單

1. **確認腳本類型**：policy / beforeInsert / beforeUpdate / function / approval / cron / batchPolicy / todoJob
2. **建立測試檔案**：在 `tests/<tableName>/` 下建立 `<scriptName>.test.ts`，function 類型放在 `functions/` 子資料夾
3. **建立或擴充 mock 資料**：在 `tests/<tableName>/mocks.ts` 定義 mock 記錄
4. **設定 `generateExecuteFunc`**：
   - `scriptPath`：指向 `tables/` 下的腳本 `.ts` 檔案路徑
   - `libs`：列出腳本依賴的共用函式庫（查看腳本的表格定義中的 `libs` 設定）
5. **設定 `MySandbox.queryProcessor`**：
   - 在 `before()` 中覆寫，在 `after()` 中還原
   - 處理所有腳本會發出的查詢（未處理的查詢應拋出錯誤）
6. **在 `beforeEach()` 中重置 mock 資料**：使用 `cloneDeep()` 確保測試間獨立
7. **撰寫測試案例**：
   - 調整 mock 資料 → 執行 → 用 `expect` 斷言結果
   - Policy：驗證回傳 `null`（通過）或包含特定錯誤訊息
   - Hook：驗證回傳的記錄欄位值
   - Function：驗證回傳值或透過 QueryEventEmitter 驗證副作用
8. **執行測試**：`npm test -- tests/<tableName>/<scriptName>.test.ts`

---

## 10. 常見陷阱與注意事項

### 10.1 必須先 build

測試前需要 `npm run build`（webpack 編譯），`generateExecuteFunc` 讀取的是編譯後的 JS 檔案。`bin/test.sh` 已自動處理，但如果單獨跑 mocha 會失敗。

### 10.2 queryProcessor 必須還原

```typescript
// 正確做法
const originQueryProcessor = MySandbox.queryProcessor;
before(() => { MySandbox.queryProcessor = ... });
after(() => { MySandbox.queryProcessor = originQueryProcessor; });

// 錯誤做法：忘記還原，會影響其他測試
```

### 10.3 cloneDeep 避免測試間汙染

```typescript
// 正確做法
let testData = cloneDeep(mockData);
beforeEach(() => { testData = cloneDeep(mockData); });

// 錯誤做法：直接修改 mock 原始物件
```

### 10.4 未處理的查詢應拋出錯誤

在 `queryProcessor` 的最後加上 `throw new Error('未處理的查詢: ' + JSON.stringify(query))`，這能幫助你快速發現遺漏的 mock。

### 10.5 call 方法的回傳值需要 JSON.stringify

```typescript
if (query.method === 'call') {
  return JSON.stringify({ result: 'ok' });  // 必須 stringify
}
```

### 10.6 逾時設定

- 預設逾時 30 秒（`-t 30000`）
- 超過 5 秒視為慢測試（`-s 5000`）
- 如果 TodoJob 需要更多時間，考慮簡化 mock 資料而非調高逾時

### 10.7 平行執行注意事項

Mocha 使用 `--parallel --jobs 2` 執行。每個測試檔案在獨立的 worker 中運行，因此 `MySandbox.queryProcessor` 的覆寫不會跨檔案影響。但同一檔案內的測試仍共用狀態，需注意 `beforeEach` 重置。

### 10.8 全域變數

`tests/global-setup.ts` 注入了 `global.dayjs`、`global.Utils`、`global.Procurement`。如果你的腳本依賴其他全域變數，需要在此檔案中補充。

### 10.9 libs 的對應關係

腳本使用的 `libs` 必須與表格定義中的設定一致。例如，如果表格定義中的腳本配置了 `libs: ['dayjs', 'Utils']`，測試時 `generateExecuteFunc` 的 `libs` 也需要包含這兩個。
