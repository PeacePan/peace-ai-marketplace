# QueryProcessor Mock 與事件追蹤

## 基本概念

`MySandbox.queryProcessor` 是一個**全域的非同步回呼函式**，所有在 VM 沙盒中透過 `ctx.query.*` 發出的資料庫操作都會被路由到這裡。測試時覆寫它來攔截並模擬查詢結果。

## 標準模式

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
          return 1;
        }
      }
      if (query.method === 'insertV2') {
        return 'new-id-001';
      }
      if (query.method === 'batchInsertV2') {
        return query.inserts.map((_, i) => `batch-id-${i}`);
      }
      if (query.method === 'call') {
        if (query.name === '我的函式') return JSON.stringify({ result: 'ok' });
      }
      if (query.method === 'echo') return;
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

## 各 query method 的回傳值規範

| method | 回傳型別 | 說明 |
|--------|---------|------|
| `find` / `findV2` | `Record[]` | 回傳記錄陣列 |
| `insert` / `insertV2` | `string` | 新建記錄的 ID |
| `batchInsertV2` | `string[]` | 多筆新建記錄的 ID 陣列 |
| `update` / `updateV2` | `number` | 影響的記錄數 |
| `batchUpdateV2` | `number` | 影響的記錄數 |
| `dangerBatchUpdateV2` | `number` | 影響的記錄數（無條件批次更新） |
| `call` | `string` (JSON) | 函式回傳值（**必須 JSON.stringify**） |
| `echo` | `void` | 無需回傳 |
| `archiveV2` / `unarchiveV2` | `number` | 影響的記錄數 |
| `batchArchiveV2` / `batchUnarchiveV2` | `number` | 影響的記錄數 |
| `todoJobsV2` | `TodoJob[]` | 待辦工作陣列 |
| `setTodoJob` | `void` | 無需回傳 |
| `publishV2` | `void` | 無需回傳 |
| `http.json` | `any` | HTTP 請求的 JSON 回應 |
| `athena` | `any[]` | Athena 查詢結果 |
| `move` | `number` | 影響的記錄數 |
| `docdbFind` | `any[]` | DocumentDB 查詢結果 |

## 進階模式：動態回傳

根據查詢條件動態回傳不同資料：

```typescript
MySandbox.queryProcessor = async (query): Promise<unknown> => {
  if (query.method === 'findV2') {
    if (query.table === 'possale') {
      // 根據查詢條件回傳不同資料
      if (query.filter?.status === 'COMPLETED') return [completedSale];
      if (query.filter?.status === 'CANCELLED') return [];
      return [testSale];
    }
  }
};
```

## 進階模式：計數與記錄

追蹤查詢被呼叫的次數和內容：

```typescript
let insertedRecords: SafeRecord2[] = [];
let batchInsertCount = 0;

beforeEach(() => {
  insertedRecords = [];
  batchInsertCount = 0;
});

MySandbox.queryProcessor = async (query): Promise<unknown> => {
  if (query.method === 'batchInsertV2' && query.table === 'mytable') {
    batchInsertCount++;
    insertedRecords.push(...query.inserts);
    return query.inserts.map((_, i) => `batch-id-${i + 1}`);
  }
};

// 在測試中驗證
it('應該批次插入記錄', async () => {
  await executeFunc({ args: [testInput, null] });
  expect(batchInsertCount).to.equal(2);
  expect(insertedRecords).to.have.lengthOf(100);
});
```

## 進階模式：beforeEach 中覆寫

當不同測試需要不同的 queryProcessor 時：

```typescript
const defaultQueryProcessor: typeof MySandbox.queryProcessor = async (query) => {
  eventEmitter.emit(query.method, query);
  if (query.method === 'findV2') return [];
  throw new Error('未處理的查詢: ' + JSON.stringify(query));
};

beforeEach(() => {
  testData = cloneDeep(mockData);
  MySandbox.queryProcessor = defaultQueryProcessor;  // 每次重置為預設
});

it('特殊情境', async () => {
  // 此測試需要不同的 mock
  MySandbox.queryProcessor = async (query) => {
    if (query.method === 'findV2') return [specialRecord];
    return defaultQueryProcessor(query);
  };
  // ...
});
```

---

## QueryEventEmitter 事件追蹤

當你需要**驗證腳本是否發出了預期的查詢**（而非只看回傳值），使用 `QueryEventEmitter`。

### 基本用法

```typescript
import { QueryEventEmitter } from '../utils';

const eventEmitter = new QueryEventEmitter();

before(() => {
  MySandbox.queryProcessor = async (query): Promise<unknown> => {
    // 每個查詢都 emit 事件
    eventEmitter.emit(query.method, query);
    // ... 回傳 mock 資料
  };
});

it('應該更新指定表格', async () => {
  let hasUpdated = false;

  eventEmitter.addListener('updateV2', (query) => {
    hasUpdated ||= query.table === 'targetTable' && query.body?.status === 'DONE';
  });

  await executeFunc({ args: [testData] });

  eventEmitter.removeAllListeners('updateV2');  // 清理
  expect(hasUpdated).to.be.true;
});
```

### 支援的事件名稱

所有 `query.method` 都可以作為事件名稱監聽，每個事件會傳入對應型別的 query 物件：

| 事件名稱 | Query 型別 |
|---------|-----------|
| `echo` | `EchoQuery` |
| `find` | `FindQuery` |
| `findV2` | `FindV2Query` |
| `insert` | `InsertQuery` |
| `insertV2` | `InsertV2Query` |
| `batchInsertV2` | `BatchInsertV2Query` |
| `update` | `UpdateQuery` |
| `updateV2` | `UpdateV2Query` |
| `batchUpdateV2` | `BatchUpdateV2Query` |
| `dangerBatchUpdateV2` | `DangerBatchUpdateV2Query` |
| `call` | `CallQuery` |
| `archiveV2` | `ArchiveV2Query` |
| `unarchiveV2` | `UnarchiveV2Query` |
| `todoJobsV2` | `TodoJobsV2Query` |
| `setTodoJob` | `SetTodoJobQuery` |
| `publishV2` | `PublishV2Query` |
| `http.json` | `FetchQuery` |
| `athena` | `AthenaQuery` |
| `move` | `MoveQuery` |
| `docdbFind` | `DocDbFindQuery` |

### 注意事項

- `QueryEventEmitter` 預設 `maxListeners` 為 20（高於 Node.js 預設的 10），避免在多測試案例時出現 memory leak 警告
- 每個測試結束後記得 `removeAllListeners()` 或 `removeListener()` 清理
- 使用 `||=` 運算子可以簡化「是否至少呼叫一次」的判斷
