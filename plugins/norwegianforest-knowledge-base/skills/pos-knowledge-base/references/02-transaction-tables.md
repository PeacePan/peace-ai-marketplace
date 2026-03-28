# 核心交易表格

## possale — 銷售單（v173）

### 基本資訊

- **表格代碼**：`POS_SALE`
- **自動編碼**：`{storeName前綴}YYMMDD[4碼]`（例：001240315-0001）
- **jobQueues**：`'1,1,1,1,1,1,1,1,1,1'`（10個 queue）
- **欄位分組**：BASIC / INVOICE / ACCOUNTING / DELIVERY / RECONCILIATION / SYSTEM

### 核心欄位

```typescript
// 基本資訊
name              // 銷售編號（自動）
channel           // 通路（ENUM: PosChannel - STORE/EMPLOYEE/WEBSITE/DELIVERY）
memberName        // 會員編號（REF → posmember）
userName          // 使用者
salerName         // 銷售員
status            // 狀態（ENUM: PosStatus - 未完成/已付款/作廢/退款中/已退款）
storeName         // 門市編號（REF → posstore）
configName        // 機台編號（REF → posconfig）
reportName        // 日結單編號（REF → posreport）

// 金額
saleTotal         // 銷售小計（料件金額 - 促銷折扣）
discount          // 折抵金額（整單折扣/折扣碼/禮券/點數折現）
total             // 總計（saleTotal - discount）
paidAmount        // 已付金額（= total + overflow，有溢收時）
paidAt            // 付款時間

// 禮券
totalWithoutVoucher  // 扣除禮券前總計
voucherTotal         // 禮券折抵金額

// 發票（INVOICE 分組）
invoiceType       // 帳務別（電子發票/收據/員購）
eGUINo            // 電子發票號碼
eGUICarrier       // 發票載具（手機條碼/自然人憑證/平台）
donationCode      // 捐贈碼
eGUIUniNo         // 統一編號（公司戶）
invoiceId         // 發票 ID
eGUICanceledAt    // 發票作廢時間

// 帳務（ACCOUNTING 分組）
totalTax          // 總稅額（= total - total/1.05，BeforeInsert 自動計算）
nsTranid          // NetSuite 交易 ID
nsSyncAt          // NetSuite 同步時間
nsJournalId       // NetSuite Journal ID
emSyncAt / emSyncStatus  // Emarsys 同步

// 外送（DELIVERY 分組）
isDelivery        // 是否外送單
deliveryName      // 收件人
deliveryMobile    // 收件人電話
deliveryAddress   // 收件地址
deliveryType      // 外送類型（ENUM: PosDeliveryType）
deliveryAnalysis  // 外送分析（ENUM）

// 外送平台
maFastBuyOrderName     // 寵速配訂單號
isFastBuyItemChanged   // 寵速配料件是否變更
uberEatsMobile         // UberEats 電話

// 客訂
customerOrderName   // 客訂單編號（REF → poscustomerorder）
originalSaleName    // 換購原銷售單
exchangeSaleName    // 換購新銷售單

// 對帳（RECONCILIATION 分組）
arrivedAt           // 對帳到帳時間
arrivedAmountTotal  // 對帳到帳金額
arriveStatus        // 對帳狀態
transferAmountTotal // 轉帳金額
handlingFeeTotal    // 手續費
```

### 表身（LineSchema）

```typescript
// items 表身（料件明細）
lineSchema: {
  items: {
    itemName: REF → ITEM,
    amount: 數量,
    price: 單價,
    subtotal: 小計,
    promotionName: 促銷編號（REF → pospromotionsnapshot）,
    promotionLineId: 促銷表身行ID,
    // ... 更多促銷細項欄位
  },
  payments: {
    type: 付款類型（ENUM: PosSalePaymentType）,
    amount: 付款金額,
    creditNo: 信用卡末四碼,
    // ... 更多付款欄位
  },
  discounts: {
    type: 折扣類型,
    name: 折扣名稱,
    amount: 折扣金額,
    // ...
  }
}
```

### 狀態機

```
未完成 → 已付款（finishCheckout 觸發）
       → 作廢
已付款 → 退款中（posreturn 建立）
退款中 → 已退款（posreturn 完成）
```

### 重要腳本

**Policy（policy.ts）**：
- 銷售小計 / 折抵 / 總計一致性驗證
- 付款項目金額驗證
- 促銷快照比對（`Pos.checkSnapshot()`）
- 發票載具互斥驗證
- 外送單相依性檢查
- 點加金料件設定驗證

**BeforeInsert（beforeInsert.ts）**：
```typescript
// 自動計算稅額
record.totalTax = record.total - Math.round(record.total / 1.05)
```

**finishCheckout（function）**：
```typescript
// 完成結帳的核心邏輯（291行）
1. Pos.pickEGUINo()        // 電子發票取號
2. 更新 status → 已付款
3. 換購銷售雙向關聯更新
4. Pos.getApportedItems()  // 料件價值攤算
5. 發起 TodoJob:
   - posmember/完成結帳後更新會員資料
   - posmember/更新會員品類標籤（Emarsys）
   - possale/價值攤算（有促銷細項時）
   - pointpromotion/更新已兌換數量（點加金）
```

**balance（TodoJob function）**：
```typescript
// 分批扣庫（每次 50 筆）
// 支援中斷恢復（buffer 傳遞 lineId 進度）
Inventory.pushBalance(ctx, {
  tableName: 'possale',
  item: itemName,
  recordName: saleName,
  amount: -amount,   // 負數為扣庫
  location: storeName,
  date: paidAt
})
```

---

## posreturn — 退貨單（v46）

### 基本資訊

- **表格代碼**：`POS_RETURN`
- **自動編碼**：`{自訂前綴}{storeName}YYMMDD[4碼]`
- **jobQueues**：`'1,1,1'`（3個 queue）
- **Library**：Utils, Pos, MemberRights, Inventory

### 核心欄位

```typescript
name            // 退貨單編號（自動）
status          // 狀態（新建/退款中/已完成/已作廢）
saleName        // 原銷售單（REF → possale）
total           // 退款總計
storeName       // 門市（REF → posstore）
configName      // 機台（REF → posconfig）
salerName       // 銷售員
reportName      // 日結單（REF → posreport）

// 發票
creditNoteNo        // 折讓單號
creditNoteCreatedAt // 折讓建立時間
eGUICanceledAt      // 發票作廢時間

// 發票同步
invoiceSyncStatus   // 同步狀態
invoiceSyncAt       // 同步時間
invoiceSyncError    // 同步錯誤

// 庫存回補
inventoryStatus   // 庫存回補狀態
inventoryError    // 庫存回補錯誤
balanceStatus     // 庫存餘額回補狀態
balanceError      // 庫存餘額回補錯誤

// 對帳
refundedAt              // 退款到帳時間
refundedAmountTotal     // 退款到帳金額
refundStatus            // 退款狀態
transferAmountTotal     // 轉帳金額
handlingFeeTotal        // 手續費

// 系統整合
ocardStatus   // Ocard 狀態
nsTranid      // NetSuite 交易 ID
nsJournalId   // NetSuite Journal ID
```

### 表身（LineSchema）

```typescript
items: {
  saleItemLineId: 對應銷售單料件行ID,
  itemName: 料件（REF → ITEM）,
  amount: 退貨數量,
  price: 退貨單價,
  subtotal: 退貨小計
}

refunds: {
  type: 退款類型（ENUM: PosSalePaymentType）,
  amount: 退款金額,
  creditNo: 信用卡末四碼,
  transType: 交易類型,
  receiptNo: 收據號碼
}

reconciliationItems: {
  tradeNo: 交易號,
  paymentType: 付款類型,
  refundedAt: 退款到帳時間,
  refundedAmount: 退款金額,
  handlingFee: 手續費
}
```

### 狀態機

```
新建 → 退款中（發起退款申請）
退款中 → 已完成（退款確認）
       → 已作廢（取消退款）
```

### 重要腳本

**Policy**：
- 退款中/已完成必須有退款項目
- 退款項目總計驗證
- 信用卡退款需有信用卡資訊
- 折讓單與發票作廢互斥
- 新增時自動寫入 possale 的退貨記錄

**balance（TodoJob function）**：
```typescript
// 回補庫存
Inventory.pushBalance(ctx, {
  tableName: 'posreturn',
  amount: +amount,  // 正數為回補
  // ...
})
```

---

## posreport — 日結單（v10）

### 基本資訊

- **表格代碼**：`POS_REPORT`
- **自動編碼**：`{storeName前綴}YYYYMMDD[4碼]`
- **jobQueues**：`'1,1,1,1,1,1,1,1,1,1'`（10個 queue）

### 核心欄位

```typescript
name             // 日結單編號（自動）
storeName        // 門市（REF → posstore）
configName       // 機台（REF → posconfig）
openingUserName  // 開帳人員
squaringUserName // 清帳人員
squaredAt        // 清帳時間
tillMoney        // 庫錢金額
retentionMoney   // 留存金額
memo             // 備註
accountingDate   // 帳務日（TIME_GRANULARITY: DAY, timezone: +8）
```

### 重要腳本

**Policy**：
- 清帳人員與清帳時間必須同時存在
- **新增時**自動更新 `posconfig.reportName`（代表開帳）

**BeforeInsert**：
- 自動關聯 posconfig，設定 `configName.reportName = name`

**closeEntry（Function）**：
- 執行日結帳務（`isPublic: true`）

---

## posconfig — 收銀機設定（v3）

### 基本資訊

- **表格代碼**：`POS_CONFIG`
- **自動編碼**：`{3位門市號}{2位機台號}`（5位數字，例：00101）

### 核心欄位

```typescript
store          // 門市（INSERT_REF → posstore）
no             // 機台號（2位數字）
name           // 收銀機編號（5位，自動）
storeType      // 門市類型（ENUM: PosConfigStoreType）
mode           // 機台模式（ENUM: PosConfigMode - 銷售機/查價機/美容工作機）
invoiceType    // 帳務別（ENUM: PosInvoiceType - 電子發票/收據/員購）
creditCardReaderType  // 信用卡機類型（ENUM）
reportName     // 日結單編號（READ，有值 = 已開帳）
status         // 使用狀態（defaultValue: ACTIVE）
```

### 重要腳本

**Policy（posconfigpolicy）**：
- 機台編號 = 門市編號 + 機台號（格式驗證）
- 查價機模式不能設定帳務類型
- 美容工作機帳務類型限制為「收據」
- 專櫃門市不能使用電子發票
- 門市銷售機帳務類型限制

---

## posstore — 門市主檔（v26）

### 基本資訊

- **表格代碼**：`POS_STORE`
- **自動編碼**：無（手動設定 3 位數字門市編號）
- **jobQueues**：`'1,1,1'`（3個 queue）
- **欄位分組**：BASIC / FINANCE_SETTING / SETTING / MANAGEMENT_SETTING

### 核心欄位（80+ 個，依分組）

```typescript
// BASIC 基本資訊
name            // 門市編號（3位數字）
displayName     // 門市名稱
phoneNo         // 電話
businessNo      // 統一編號
salonNo         // 美容台數
postalCode      // 郵遞區號
address         // 地址
registeredAddress // 登記地址
HCTNo           // HCT 號碼（宅配用）

// FINANCE_SETTING 財務設定
eGUIUniNo       // 電子發票統一編號
groupId         // 會計群組
departmentId    // 部門 ID
taxNo           // 稅號
eGUINoTitle     // 電子發票抬頭
edenredClientId/Secret  // 宜睿禮券設定
status          // 門市狀態（ENUM: PosStoreStatus）

// SETTING 營運設定
ocardName       // Ocard 門市名稱
storeCity/Zip/Address  // 門市位置（Ocard 用）
storeManagementDep/Area  // 門市管理部門/區域
salonManagementDep/Area  // 美容管理部門/區域
storeLocationArea  // 門市商圈
manpower        // 人力
category        // 門市類型
openDate        // 開幕日
logisticsUrl    // 物流系統 URL
longitude/latitude  // 經緯度

// MANAGEMENT_SETTING 管理設定
chiefSalonOfficer    // 美容主任
storeManager         // 店長
regionalStoreManager // 區域主任
salonSupervisor      // 美容督導
contactEmail         // 聯絡信箱
sign                 // 事業別（ENUM: PosStoreSign）
storeType            // 門市類型（ENUM）
accommodationType    // 裝潢類型（ENUM）
hasMedicamentLicense // 是否有藥品許可
toiletOpen           // 廁所是否開放
livingBody           // 生活館類型（ENUM）
hasParkingSpace      // 是否有停車位
disableCheckAmount   // 關閉數量檢查
```

### 表身（LineSchema）

```typescript
users: {
  user: 允許使用者名稱,
  memo: 備註
}

servingPetGoal: {
  month: 月份（1-12）,
  amount: 目標寵物數量
}
```

### 重要腳本

**Policy（posstore policy）**：
- `servingPetGoal` 不可重複設定相同月份
- 關閉數量檢查需同時設定開始/結束時間
- 事業別門市代碼格式驗證（A[門市編號]C）
- 人力格式驗證

**NetSuite（Function + Cron）**：
- `isPublic: true`，`onJobRetry: 3`
- HALF_HOURLY 排程
