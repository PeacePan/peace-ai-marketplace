# 各類腳本測試範例

## 1. Policy 腳本測試

Policy 腳本接收一筆記錄，驗證後回傳錯誤訊息（`string`）或 `null`/`undefined`（通過）。

```typescript
import { expect } from 'chai';
import { cloneDeep } from 'lodash';
import { resolve } from 'path';
import { MySandbox } from '@javaCat/MySandbox';
import { SafeRecord2 } from '@norwegianForestTypes/index';
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
      if (query.method === 'todoJobsV2') return [];
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

**重點**：
- Policy 回傳 `null` / `undefined` 代表通過，回傳 `string` 代表驗證失敗
- 用 `expect(result).to.include('關鍵字')` 驗證錯誤訊息（避免完全匹配，較不容易因文案調整而壞掉）

---

## 2. BeforeInsert / BeforeUpdate 腳本測試

Hook 腳本在記錄新增/修改前轉換資料。

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

**BeforeUpdate 的參數結構不同**：

```typescript
// beforeUpdate 的參數是 [updateData, originalRecord]
const executeFunc = generateExecuteFunc<[UpdateData, SafeRecord2<MyBody>]>({
  scriptPath: resolve(__dirname, '../../tables/myFolder/scripts/myTable/beforeUpdate.ts'),
  libs: [],
});

it('更新前應檢查並轉換', async () => {
  const result = await executeFunc({ args: [testUpdate, testOriginal] });
  // result 可能是修改後的 update data 或錯誤訊息字串
  expect(result).to.not.be.a('string');  // 不是字串代表通過
});
```

---

## 3. Function 腳本測試

Function 腳本的參數格式為 `[record, argument]`。通常需要較複雜的 queryProcessor mock。

```typescript
import batchProcess from '@norwegianForestTables/function/scripts/batchmutation/process';

// 利用 import 的函式推導參數型別
type FuncArgs = [Parameters<typeof batchProcess>[0], null];

const tablesRoot = resolve(__dirname, '../../../tables');
const executeFunc = generateExecuteFunc<FuncArgs>({
  scriptPath: resolve(tablesRoot, 'function/scripts/batchmutation/process.ts'),
  libs: [
    { name: 'dayjs', path: resolve(tablesRoot, 'common/dayjs.ts') },
    { name: 'Utils', path: resolve(tablesRoot, 'common/utils.ts') },
    { name: 'Validator', path: resolve(tablesRoot, 'common/validator.ts') },
  ],
});

// 追蹤副作用
let insertedRecords: SafeRecord2[] = [];

beforeEach(() => {
  insertedRecords = [];
});

before(() => {
  MySandbox.queryProcessor = async (query): Promise<unknown> => {
    if (query.method === 'findV2') {
      if (query.table === 'batchmutation') return [testBatchMutation];
      if (query.table === '__file2__') return [testFile2];
      if (query.table === '__user__') return [mockUser];
      return [];
    }
    if (query.method === 'batchInsertV2' && query.table === 'testtable') {
      insertedRecords.push(...query.inserts);
      return query.inserts.map((_, i) => `batch-id-${i + 1}`);
    }
    if (query.method === 'updateV2' && query.table === 'batchmutation') {
      Object.assign(testBatchMutation.body, query.body);
      return 1;
    }
    throw new Error('未處理的查詢: ' + JSON.stringify(query));
  };
});

it('應成功批次插入記錄', async () => {
  const result = await executeFunc({ args: [testInput, null] });
  expect(insertedRecords).to.have.lengthOf(10);
});
```

**重點**：
- 用 `Parameters<typeof importedFunc>[0]` 推導參數型別，避免手動定義
- Function 的第二個參數通常是 `null`（如果不需要 argument）
- 複雜的 function 可能需要追蹤多種副作用（insert、update、email 等）

---

## 4. Approval 腳本測試

使用 `findApprovalRule` 工具函式，不需要自行設定 `generateExecuteFunc`。

```typescript
import { expect } from 'chai';
import { findApprovalRule } from '../utils';
import { TABLE_NAME } from '@norwegianForestTables/const';
import { VendorSubsidiary } from '@norwegianForestTables/enum';
import { RequisitionType } from '@norwegianForestTables/procurement/_enum';
import { RequisitionBody } from '@norwegianForestTables/procurement/_type';
import { SafeRecord } from '@norwegianForestTypes/index';

describe('請購單簽核鏈測試', () => {
  const mockRecord: SafeRecord<Pick<RequisitionBody, 'subsidiary' | 'saleTotal' | 'type' | 'location'>> = {
    body: {
      subsidiary: null,
      saleTotal: 1,
      type: null,
    },
  };

  it('IT 請購單應走萬達國際部簽核鏈', async () => {
    mockRecord.body.subsidiary = VendorSubsidiary.WP;
    mockRecord.body.type = RequisitionType.IT;
    const ruleName = await findApprovalRule(mockRecord, TABLE_NAME.REQUISITION, 'procurement');
    expect(ruleName).to.equal('萬達國際部');
  });

  it('EC 請購單應走萬達電商商品部簽核鏈', async () => {
    mockRecord.body.subsidiary = VendorSubsidiary.WP;
    mockRecord.body.type = RequisitionType.EC;
    const ruleName = await findApprovalRule(mockRecord, TABLE_NAME.REQUISITION, 'procurement');
    expect(ruleName).to.equal('萬達電商商品部');
  });
});
```

**重點**：
- `findApprovalRule` 內部自動處理 Sandbox 執行
- 第三個參數 `folder` 是 `tables/` 下的資料夾名稱
- 不需要設定 `queryProcessor`（approval 腳本通常不查詢資料庫）

---

## 5. Cron 腳本測試

Cron 腳本通常會 export 純函式，可以**直接 import 測試**，不需要走 VM Sandbox。

```typescript
import { expect } from 'chai';
import { checkDateIsValidToAddWeekBillNote } from '@norwegianForestTables/procurement/scripts/vendor/note.cron';
import dayjs from '@norwegianForestTables/common/dayjs';

describe('廠商檔 - 週對帳通知單觸發時間', () => {
  it('週對帳通知單只在週一會觸發', () => {
    const monday = dayjs('2023-01-30T00:00:00+08:00').toDate();
    expect(checkDateIsValidToAddWeekBillNote(monday)).to.be.true;
  });

  it('週二~週日不會觸發', () => {
    ['2023-01-31', '2023-02-01', '2023-02-02', '2023-02-03', '2023-02-04', '2023-02-05'].forEach((dateStr) => {
      const date = dayjs(`${dateStr}T04:00:00+08:00`).toDate();
      expect(checkDateIsValidToAddWeekBillNote(date)).to.be.false;
    });
  });
});
```

**重點**：
- Cron 腳本中 export 的純函式（不依賴 `ctx`）可以直接 import
- 測試檔案命名為 `<name>.cron.test.ts`
- 不需要 `generateExecuteFunc` 和 `queryProcessor`

---

## 6. TodoJob 函式測試

使用 `executeFuncUntilDoneOrError` 處理多階段執行。

```typescript
import { expect } from 'chai';
import { cloneDeep } from 'lodash';
import { resolve } from 'path';
import { executeFuncUntilDoneOrError, generateExecuteFunc } from 'tests/utils';
import { MySandbox } from '@javaCat/MySandbox';
import { mockTodoJob } from '../mocks';

const tablesRoot = resolve(__dirname, '../../../tables');
const executeFunc = generateExecuteFunc<MyFuncArgs>({
  scriptPath: resolve(tablesRoot, 'function/scripts/myTable/myTodoJob.ts'),
  libs: [
    { name: 'dayjs', path: resolve(tablesRoot, 'common/dayjs.ts') },
    { name: 'Utils', path: resolve(tablesRoot, 'common/utils.ts') },
  ],
});

describe('我的 TodoJob 測試', () => {
  const originQueryProcessor = MySandbox.queryProcessor;
  let processedItems: any[] = [];

  before(() => {
    MySandbox.queryProcessor = async (query): Promise<unknown> => {
      if (query.method === 'findV2') {
        if (query.table === 'mytable') return [testRecord];
        return [];
      }
      if (query.method === 'batchUpdateV2' && query.table === 'mytable') {
        processedItems.push(...query.updates);
        return query.updates.length;
      }
      throw new Error('未處理的查詢: ' + JSON.stringify(query));
    };
  });

  beforeEach(() => {
    processedItems = [];
  });

  after(() => {
    MySandbox.queryProcessor = originQueryProcessor;
  });

  it('待辦工作應執行完成', async () => {
    const result = await executeFuncUntilDoneOrError({
      executeFunc,
      args: [testInput, null],
      todoJob: cloneDeep(mockTodoJob),
    });
    expect(result.status).to.equal('DONE');
    expect(processedItems).to.have.lengthOf.greaterThan(0);
  });
});
```

**重點**：
- `todoJob` 記錄的 `body.buffer` 會在迭代間自動傳遞
- `executeFuncUntilDoneOrError` 會自動斷言最終 status
- 記得用 `cloneDeep(mockTodoJob)` 避免跨測試汙染

---

## 7. BatchPolicy 腳本測試

BatchPolicy 一次驗證多筆記錄。

```typescript
const executeFunc = generateExecuteFunc<[{ records: SafeRecord2[] }]>({
  scriptPath: resolve(__dirname, '../../tables/myFolder/scripts/myTable/batchPolicy.ts'),
  libs: [],
});

describe('批次政策測試', () => {
  // ... queryProcessor 設定 ...

  it('多筆合法記錄應通過批次驗證', async () => {
    const result = await executeFunc({
      args: [{ records: [testRecord1, testRecord2] }],
    });
    expect(result).to.be.null;
  });

  it('包含非法記錄應回傳錯誤', async () => {
    testRecord2.body.invalidField = 'bad';
    const result = await executeFunc({
      args: [{ records: [testRecord1, testRecord2] }],
    });
    expect(result).to.be.a('string');
    expect(result).to.include('錯誤關鍵字');
  });
});
```

---

## Mock 資料檔案範例

`mocks.ts` 定義測試用的假資料：

```typescript
// tests/possale/mocks.ts
import { PosSaleBody, PosSaleLines } from '@norwegianForestTables/pos/_type';
import { PosSaleStatus, PosSaleChannel } from '@norwegianForestTables/enum';
import { SafeRecord } from '@norwegianForestTypes/index';

export const mockSale: SafeRecord<PosSaleBody, PosSaleLines> = {
  body: {
    _id: 1,
    _createdAt: new Date('2023-04-19T03:11:32.664Z'),
    _createdBy: 'token:mock99900Token',
    name: '999202304190001',
    storeName: '999',
    status: PosSaleStatus.OPEN,
    channel: PosSaleChannel.POS,
    total: 1000,
    // ... 其他必填欄位
  },
  lines: {
    items: [/* 商品明細 */],
    promotions: [/* 促銷明細 */],
    payments: [/* 付款明細 */],
  },
};

export const mockStore: SafeRecord<Pick<PosStoreBody, 'name' | 'displayName'>> = {
  body: {
    name: '999',
    displayName: '測試門市',
  },
};
```

**重點**：
- 使用 `Pick<Body, 'field1' | 'field2'>` 只定義測試需要的欄位
- 匯出為 `const`，測試中再用 `cloneDeep()` 複製
- 一個表格的所有測試共用同一份 `mocks.ts`
