# TodoJob 與 Cron 排程模式

## POS 系統所有 Cron 排程清單

| 表格 | 排程名稱 | 頻率 | 是否 useTodoJob | 功能 |
|-----|---------|-----|----------------|------|
| `possale` | 扣庫v2 | TEN_MINS | ✅ | 銷售後扣除庫存 |
| `posreturn` | 回補庫存v2 | TEN_MINS | - | 退貨後回補庫存 |
| `posdeposit` | 回補庫存排程 | FIVE_MINS | - | 寄貨單庫存回補 |
| `posdepositwithdraw` | 扣庫排程 | FIVE_MINS | ✅ | 寄貨提領扣庫 |
| `poscustomerorderwithdraw` | 扣庫排程 | - | - | 客訂提領扣庫 |
| `poscustomerorderwithdraw` | 補拋NS排程 | - | - | NetSuite 補發 |
| `poscustomerorder` | replenishInventory | - | - | 客訂到貨補貨 |
| `poscustomerorder` | balanceReductionAfterCancellation | - | - | 取消後扣庫 |
| `poscustomerorder` | addToStoreDailyReorderList | - | - | 加入補貨清單 |
| `pospromotion` | 活動料件集合檔刷新 | HALF_HOURLY | - | 促銷料件集合同步 |
| `pospromotion` | 門市上限使用數量刷新 | HALF_HOURLY | - | 刷新門市已用量 |
| `pospromotion` | 同步Foodomo售價 | 定時 | - | Foodomo 整合 |
| `poscoupon` | 活動料件集合檔刷新 | HALF_HOURLY | - | 折扣碼料件同步 |
| `posorderdiscount` | 活動料件集合檔刷新 | HALF_HOURLY | - | 整單折扣料件同步 |
| `posorderdiscount` | 門市上限使用數量刷新 | HALF_HOURLY | - | 刷新門市已用量 |
| `posfreebie` | 門市上限使用數量刷新 | HALF_HOURLY | ✅（isPublic） | 刷新門市已用量 |
| `posvoucher` | 活動料件集合檔刷新 | DAILY（UTC 04:00） | - | 禮券料件同步 |
| `posvoucherlog` | 扣庫v2 | TEN_MINS | - | 禮券兌換扣庫 |
| `poscouponcodegenerator` | 產生序號 | - | ✅（isPublic） | 批次建立折扣序號 |
| `posstore` | NetSuite | HALF_HOURLY | ✅（onJobRetry: 3） | 門市資料同步 |
| `positemcollection` | syncItems | 定時 | - | 料件同步 |
| `positemcollection` | moveOldData | 定時 | - | 搬移舊資料 |
| `positemcollection` | posFreebieItemCollection | 定時 | - | 加贈料件集合 |

---

## CronRate 常數對照

```typescript
// 在表格定義中使用
import { CronRate } from '@my-app/lib'

CronRate.FIVE_MINS   // 每 5 分鐘
CronRate.TEN_MINS    // 每 10 分鐘
CronRate.HALF_HOURLY // 每 30 分鐘
CronRate.HOURLY      // 每小時
CronRate.DAILY       // 每日
```

---

## POS 系統的 TodoJob 模式

### 模式一：分批扣庫（possale/balance.ts）

```typescript
// 最典型的分批 TodoJob 模式
const cChunkRowsPerTodoJob = 50  // 每次處理 50 筆

async function balance(ctx, params) {
  const { saleName } = params
  const buffer = params.buffer || { processedLineIds: [] }  // 斷點恢復

  // 讀取銷售單料件
  const sale = await ctx.query.find('possale', { filter: { name: saleName } })
  const pendingItems = sale.items.filter(
    item => !buffer.processedLineIds.includes(item._id)
  )

  // 分批處理
  const chunkItems = pendingItems.slice(0, cChunkRowsPerTodoJob)

  for (const item of chunkItems) {
    await Inventory.pushBalance(ctx, {
      tableName: 'possale',
      item: item.itemName,
      recordName: saleName,
      amount: -item.amount,  // 負數為扣庫
      location: sale.storeName,
      date: sale.paidAt,
      lineId: item._id  // 防止重複扣庫
    })
    buffer.processedLineIds.push(item._id)
  }

  // 若還有剩餘，繼續下一批（WORKING 狀態）
  if (pendingItems.length > cChunkRowsPerTodoJob) {
    return { status: 'WORKING', buffer }
  }

  return { status: 'DONE' }
}
```

### 模式二：防止並行衝突（queueNo 設計）

```typescript
// 使用 queueNo 確保同一目標不會並行處理
const queueNo = Utils.generateTodoJobQueueNo({
  targetName: memberName,  // 以會員為隔離單位
  queryCount: 20           // 20 個 queue slot
})

// 同一 memberName 的 TodoJob 會排在同一個 queue，按序處理
// 不同 memberName 的 TodoJob 可能並行（分到不同 queue）
```

### 模式三：料件集合同步（positemcollection/syncItemsTodoJob.ts）

```typescript
// 大批量料件查詢（可能上萬筆）
async function syncItemsTodoJob(ctx, params) {
  const { eventName } = params

  // 超時控制（48秒）
  const result = await executeFunctionWithTimeout(async () => {
    // 從促銷活動讀取篩選器
    const filters = await getEventFilters(ctx, eventName)

    // 大批量查詢料件（chunkSize: 30000）
    const matchedItems = await Procurement.findItemsForItemCollection(ctx, {
      filters,
      chunkSize: 30000
    })

    // 比對現有集合並更新
    const currentItems = await getCurrentItems(ctx, eventName)
    const toAdd = matchedItems.filter(i => !currentItems.includes(i))
    const toRemove = currentItems.filter(i => !matchedItems.includes(i))

    if (toAdd.length > 0) await ctx.query.update('positemcollection', { $push: ... })
    if (toRemove.length > 0) await ctx.query.update('positemcollection', { $pull: ... })

  }, 48)  // 48秒超時

  return result  // TodoJobFunctionReturn
}
```

### 模式四：使用 writeConflictAutoRetry

```typescript
// 高並發場景下，多個 TodoJob 同時寫入同一記錄時使用
await writeConflictAutoRetry(async () => {
  await ctx.query.update('posmember', { ... })
})
// 若發生寫入衝突（版本號不符），自動重試
```

---

## TodoJob 函式撰寫注意事項

### 1. targetName 對應

```typescript
// TodoJob 的 targetName 決定放入哪個 queueNo
// POS 系統常見的 targetName：
{ targetName: saleName, queryCount: 10 }    // 以銷售單隔離
{ targetName: memberName, queryCount: 20 }  // 以會員隔離
{ targetName: storeName, queryCount: 5 }    // 以門市隔離
```

### 2. session: true 確保冪等性

```typescript
// 若函式執行可能重複，使用 session: true
// 系統會在資料庫層面確保相同的 TodoJob 只執行一次
{
  functionName: '扣庫v2',
  session: true,  // 冪等性保護
  tableName: 'possale',
  name: saleName
}
```

### 3. buffer 跨階段傳遞

```typescript
// buffer 用於在 WORKING → WORKING 之間傳遞進度
// 注意：buffer 必須是可序列化的（無法傳遞 function、class instance）

return {
  status: 'WORKING',
  buffer: {
    processedLineIds: [...已處理的 lineId],
    lastProcessedIndex: 50
  }
}
```

### 4. keepQueue 流控

```typescript
// keepQueue 控制是否繼續佔用同一 queue
// POS 系統通常使用預設值（false），讓後續 TodoJob 可以插入

// 若某個 function 需要獨占 queue（如批次扣庫），設定：
keepQueue: true  // 確保下一批繼續在同一 queue 執行
```

---

## Cron 建立的關鍵知識

### 在表格定義中建立 Cron

```typescript
// tables/pos/possale.ts 中
const mySale: MyTableRecord = {
  ...
  functions: [
    {
      name: '扣庫v2',
      type: FunctionType.TODOJOB,
      isPublic: true,
      onJobRetry: 3,
    }
  ],
  crons: [
    {
      name: '扣庫v2排程',
      functionName: '扣庫v2',
      rate: CronRate.TEN_MINS,
      useTodoJob: true,  // 透過 TodoJob 機制執行
    }
  ]
}
```

### useTodoJob 的差異

| useTodoJob | 執行方式 | 適用場景 |
|-----------|---------|---------|
| `false`（預設） | Lambda 直接呼叫函式 | 輕量、快速的定時任務 |
| `true` | 建立 TodoJob 記錄，Worker 撿起執行 | 耗時長、需要重試、需要進度追蹤的任務 |

### 建立新排程的步驟

1. 在表格定義的 `functions` 中新增函式（含 `isPublic: true`）
2. 在表格定義的 `crons` 中新增排程
3. 在 `scripts/{tableName}/{functionName}.ts` 撰寫函式邏輯
4. 遞增表格 `version`
5. 為新函式撰寫測試（參考 `norwegianforest-workflow:norwegianforest-testing-architecture`）

---

## 發起 TodoJob 的標準做法

```typescript
// 在任何腳本中（policy/function/cron 皆可）發起 TodoJob
await ctx.query.call('todojob', 'insert', {
  body: {
    tableName: 'possale',
    functionName: '扣庫v2',
    name: saleName,         // 作為 TodoJob 的識別名稱
    param: { saleName },    // 傳給函式的參數
    queueNo: Utils.generateTodoJobQueueNo({
      targetName: saleName,
      queryCount: 10
    }),
    session: true           // 確保冪等性（可選）
  }
})
```
