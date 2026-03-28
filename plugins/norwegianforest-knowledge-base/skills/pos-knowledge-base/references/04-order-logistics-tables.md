# 訂單物流表格

## poscustomerorder — 客訂單（v42）

### 基本資訊

- **表格代碼**：`POS_CUSTOMER_ORDER`
- **自動編碼**：`CO{storeName前綴}YYMMDD[3碼]`（例：CO00124031500001）
- **jobQueues**：`'1,1,1,1,1'`（5個 queue）
- **Library**：Utils, dayjs, Inventory, Procurement

### 客訂單使用場景

當顧客購買的商品門市無庫存時，可建立客訂單預訂。系統會：
1. 自動建立調撥單（TransferOrder）調貨
2. 到貨後建立客訂提領單（poscustomerorderwithdraw）
3. 待客人來取貨時完成提領

### 核心欄位

```typescript
name              // 客訂單編號（自動）
saleName          // 對應銷售單（REF → possale）
storeName         // 門市（REF → posstore）
status            // 狀態（新建/待調貨/調貨中/待提領/已完成/已取消）
memberName        // 會員（REF → posmember）
contactName       // 聯絡人姓名
contactPhone      // 聯絡電話
contactMethod     // 聯絡方式
salesPerson       // 銷售員
memo              // 備註
needPrintMemo     // 需要列印備註（BOOLEAN）
incompleteTransfer  // 是否有未完成調撥（BOOLEAN）

// 庫存狀態
balanceStatus           // 扣庫狀態（待處理/處理中/完成/錯誤）
balanceError            // 扣庫錯誤
postCancelBalanceStatus  // 取消後回補狀態
postCancelBalanceError   // 取消後回補錯誤

// NetSuite 整合
nsCashSaleId / nsCashSaleSyncAt   // 銷貨單同步
nsCashRefundId / nsCashRefundSyncAt  // 退款單同步
```

### 表身（LineSchema）

```typescript
items: {
  itemName          // 料件（REF → ITEM）
  amount            // 訂購數量
  withdrawAmount    // 已提領數量（READ_NULLABLE）
  warehouseAmount   // 倉庫到貨數量（READ_NULLABLE）
  originalWarehouseAmount // 原始倉庫數量
  preOrderAmount    // 預訂數量
}

transferDetails: {
  // 調撥明細
  itemName    // 料件
  type        // 調撥類型
  recordName  // 調撥單號
  storeName   // 來源門市
  amount      // 調撥數量
  receivedAmount // 已收到數量
  removed     // 是否已移除
  memo        // 備註
}

memoHistory: {
  // 備註歷史
  content     // 備註內容
  salesPerson // 銷售員
}
```

### 重要腳本

**Policy（150+行）**：
- 銷售單存在性與狀態驗證
- 客訂單與寄貨單互斥（同一銷售單不能同時有客訂和寄貨）
- XXXX 會員不開放客訂
- 僅 STORE 通路可建立客訂單
- 料件有效性與數量驗證（不超過購買數）
- 自動建立調撥單（TransferOrder）

**BeforeInsert/BeforeUpdate Hook**：
- 初始化表身欄位預設值

**Cron（3個）**：
- `replenishInventory`：到貨補充庫存
- `balanceReductionAfterCancellation`：取消後扣庫
- `addToStoreDailyReorderList`：加入門市每日補貨清單

---

## poscustomerorderwithdraw — 客訂提領單（v19）

### 基本資訊

- **表格代碼**：`POS_CUSTOMER_ORDER_WITHDRAW`
- **自動編碼**：`CW{storeName前綴}YYMMDD[3碼]`
- **jobQueues**：`'1'`（1個 queue）
- **Library**：Utils, Inventory, dayjs

### 核心欄位

```typescript
name               // 提領單編號（自動）
customerOrderName  // 對應客訂單（REF → poscustomerorder）
storeName          // 門市（REF → posstore）
salesPerson        // 銷售員
memo               // 備註

// 庫存狀態
balanceStatus  // 扣庫狀態
balanceError   // 扣庫錯誤

// NetSuite
nsJournalId / nsJournalSyncAt   // Journal 同步
nsCashSaleId / nsCashSaleSyncAt // 銷貨單同步
```

### 表身（LineSchema）

```typescript
items: {
  itemName  // 料件（REF → ITEM）
  amount    // 提領數量
}
```

### 重要腳本

**Policy**：
- 客訂單狀態驗證
- 提領數量不超過剩餘可提領數量

**Cron（2個）**：
- `扣庫排程`：執行庫存扣除
- `補拋NS排程`：補發 NetSuite 同步

---

## posdeposit — 寄貨單（v13）

### 基本資訊

- **表格代碼**：`POS_DEPOSIT`
- **自動編碼**：`DE[6碼]`
- **jobQueues**：`'1'`（1個 queue）

### 寄貨單使用場景

顧客購買商品後，若無法立即帶走（如寵物飼料太重），可建立寄貨單請門市代為保管，之後再來提領。

### 核心欄位

```typescript
name      // 寄貨單編號（自動）
saleName  // 對應銷售單（REF → possale）
storeName // 門市（REF → posstore）
status    // 狀態（ENUM: DepositStatus - 新建/寄貨中/已完成）
memo      // 備註

// 庫存
balanceStatus  // 庫存回補狀態
balanceError   // 庫存回補錯誤
```

### 表身（LineSchema）

```typescript
items: {
  lineReadWrite: INSERT,  // 只有新增時可填寫
  scriptReadWrite: UPDATE, // 腳本可更新
  itemName       // 料件（REF → ITEM）
  amount         // 寄貨數量（min=1）
  withdrawAmount // 已提領數量（READ_NULLABLE）
}
```

### 重要腳本

**Policy**：
- 銷售單必須存在且已付款（paidAt 不為空）
- XXXX 會員不開放寄貨
- 員購通路不開放寄貨
- 客訂單與寄貨單互斥（同一銷售單）
- 料件必須存在於銷售單中
- 寄貨數量不超過（購買數 - 已退貨數）
- 新建時：
  - 自動設定 status = 新建
  - 初始化 `items[].withdrawAmount = 0`
  - 帳務日期驗證（日結單建立時間不超過 21 小時前）

**BATCH_BEFORE_INSERT Hook**：
- 批次新增前驗證多筆寄貨單料件不重複

**balance（Function + Cron）**：
- 庫存回補（回補被銷售單扣掉的庫存）
- `isPublic: true`
- FIVE_MINS 排程

---

## posdepositwithdraw — 寄貨提領單（v8）

### 基本資訊

- **表格代碼**：`POS_DEPOSIT_WITHDRAW`
- **自動編碼**：`DW[6碼]`
- **jobQueues**：`'1'`；CronRate: `FIVE_MINS`

### 核心欄位

```typescript
name          // 提領單編號（自動）
depositName   // 寄貨單（REF → posdeposit）
storeName     // 門市（READ，腳本自動從 posdeposit 抓取）

// 庫存
balanceStatus  // 庫存回補狀態
balanceError   // 庫存回補錯誤
```

### 表身（LineSchema）

```typescript
items: {
  itemName  // 料件（REF → ITEM）
  amount    // 提領數量（INT，min=1）
}
```

### 重要腳本

**Policy（posdepositwithdrawpolicy）**：
- 寄貨單狀態與存在性驗證
- 提領數量不超過剩餘可提領量

**扣庫（Function + Cron）**：
- `isPublic: true`
- FIVE_MINS 排程，`useTodoJob: true`

---

## posvoucherlog — 禮券兌換紀錄（v9）

### 基本資訊

- **表格代碼**：`POS_VOUCHER_LOG`
- **自動編碼**：`[流水號8碼]`（initialNumber: 1，autoKeyMinDigits: 8）
- **jobQueues**：`'1,1,1,1'`（4個 queue）

### 核心欄位

```typescript
name         // 兌換編號（自動）
voucherName  // 禮券活動（REF → posvoucher）
amount       // 兌換金額
memberName   // 會員（REF → posmember）
userName     // 使用者
eGUINo       // 電子發票號碼
reportName   // 日結單（REF → posreport）
salerName    // 銷售員
storeName    // 門市（REF → posstore）
configName   // 機台（REF → posconfig）

// 庫存
inventoryStatus / inventoryError  // 庫存狀態
balanceStatus / balanceError      // 庫存餘額狀態

// NetSuite
nsTranid    // NetSuite 交易 ID
nsSyncAt    // NetSuite 同步時間
```

### 表身（LineSchema）

```typescript
items: {
  itemName  // 料件（REF → ITEM）
  amount    // 數量
}
```

### 重要腳本

**Policy**：
- 禮券活動存在性與有效期驗證
- 兌換金額驗證

**扣庫v2（Function + Cron）**：
- `isPublic: true`
- TEN_MINS 排程
