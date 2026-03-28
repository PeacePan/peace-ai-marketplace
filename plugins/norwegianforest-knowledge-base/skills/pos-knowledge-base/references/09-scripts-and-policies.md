# Policy / Hook 腳本業務邏輯摘要

## Lifecycle Hook 完整流程

```
INSERT 流程：
  客戶端送出新增請求
      ↓
  batchPolicy INSERT 情境（oldRecords = undefined）
      ↓
  batchBeforeInsert（若定義）
      ↓
  policy（單筆）
      ↓
  beforeInsert（若定義）
      ↓
  寫入 MongoDB

UPDATE 流程：
  客戶端送出更新請求
      ↓
  batchPolicy UPDATE 情境（oldRecords = 舊值陣列）
      ↓
  batchBeforeUpdate（若定義，注意：無法被 ignorePolicy 跳過）
      ↓
  policy（單筆）
      ↓
  beforeUpdate（若定義）
      ↓
  寫入 MongoDB
```

> **重要**：`ignorePolicy: true` 只跳過 `policy` 和 `batchPolicy`，不跳過 `beforeInsert`、`beforeUpdate`、`batchBeforeInsert`、`batchBeforeUpdate`。

---

## 各表格 Policy 業務規則摘要

### possale — 銷售單 Policy

**核心驗證（按重要性排序）**：

```typescript
// 1. 金額一致性
saleTotal === Σ(items 料件金額 - 促銷折扣)
discount === Σ(整單折扣 + 折扣碼 + 禮券 + 點數折現)
total === saleTotal - discount
paidAmount === total + overflow（溢收）

// 2. 付款項目驗證
paidAmount === Σ(payments[].amount)
信用卡付款需有 creditNo
UberEats/外送平台付款需有相關憑證

// 3. 快照比對
Pos.checkSnapshot('pospromotion', promotionNames)
Pos.checkSnapshot('poscoupon', couponNames)
Pos.checkSnapshot('posorderdiscount', orderDiscountNames)

// 4. 發票驗證
invoiceType === '電子發票' → eGUINo 必須存在
載具互斥：手機條碼/自然人憑證/平台 三選一
公司戶：eGUIUniNo 必須存在

// 5. 外送驗證
isDelivery === true → 收件資料必須完整
外送單不能是員購通路
```

### posreturn — 退貨單 Policy

```typescript
// 退款中/已完成 必須有退款項目
status IN ['退款中', '已完成'] → refunds.length > 0

// 退款金額驗證
total === Σ(refunds[].amount)
Σ(refunds[].amount) <= possale.paidAmount（不能超過已付金額）

// 信用卡退款
若 refunds[].type === '信用卡' → creditNo 必須存在

// 發票互斥
creditNoteNo（折讓單）與 eGUICanceledAt（發票作廢）不能同時存在

// 新增時自動更新
UPDATE possale（寫入退貨記錄）
使用 PosUserContent.skipReturnUpdate 防迴圈
```

### posreport — 日結單 Policy

```typescript
// 清帳驗證
squaringUserName 與 squaredAt 必須同時存在或同時為空

// 新增時關聯機台
INSERT → UPDATE posconfig.reportName = posreport.name
使用 PosUserContent.skipConfigUpdate 防迴圈
```

### posconfig — 收銀機 Policy（posconfigpolicy）

```typescript
// 機台編號格式
name === store.name + no（5位數字）

// 模式限制
mode === '查價機' → invoiceType 不能設定
mode === '美容工作機' → invoiceType 必須 === '收據'

// 門市類型限制
storeType === '專櫃' → invoiceType 不能 === '電子發票'
storeType === '門市銷售機' → invoiceType 只能 === '電子發票' | '收據'
```

### posstore — 門市 Policy（posstore policy）

```typescript
// 服務目標月份不重複
servingPetGoal[].month 不能重複

// 關閉數量檢查
disableCheckAmount === true → 必須同時有 disableCheckAmountStartAt 和 disableCheckAmountEndAt

// 事業別格式
sign === '有' → salonNo 格式驗證（A[3位門市號]C）

// 人力格式
manpower 正則驗證
```

### pospromotion — 商品促銷 Policy（283行）

```typescript
// 類型限制
type === 'ITEM_PRICE' → items 表身不能為空
type === 'COMBINATION' → items 至少 2 筆
type === 'QUANTITY_DISCOUNT' → effects 表身不能為空
type === 'SECOND_ITEM_DISCOUNT' → min 必須 >= 2

// 折扣設定互斥（三選一）
percentage / directPrice / discount 只能設定一個

// effects 表身驗證（滿件折扣）
effects 必須依 min 遞增排序
每個 effect 同樣需要 三選一 折扣設定

// 建立快照
Pos.insertSnapshot(ctx, 'pospromotion', record)
```

### poscoupon — 折扣碼 Policy

```typescript
// 類型限制
type === 'ITEM' → filters 表身不能為空

// 日期設定不重複
weekly[].day 不能重複（1-7）
monthly[].date 不能重複（1-31）

// 建立快照
Pos.insertSnapshot(ctx, 'poscoupon', record)
```

### posorderdiscount — 整單折扣 Policy

```typescript
// 折扣互斥
percentage 與 discount 只能設定一個

// 日期設定不重複
weekly / monthly 同 poscoupon

// 建立快照
Pos.insertSnapshot(ctx, 'posorderdiscount', record)
```

### posfreebie — 加贈活動 Policy

```typescript
// 類型限制
type IN ['ITEM_AMOUNT', 'ITEM_QUANTITY'] → sourceItemFilters 不能為空

// 會員限量限制
memberLimitSet > 0 → target 必須 === '會員'

// 加購金額
type === 'ADDON' → addonPrice 必須 > 0

// 日期設定不重複
weekly / monthly 同 poscoupon
```

### poscustomerorder — 客訂單 Policy（150行）

```typescript
// 銷售單驗證
saleName 必須存在且 possale.status === '已付款'
possale.paidAt 必須存在

// 互斥驗證
同一 saleName 不能同時有 posdeposit 和 poscustomerorder

// 會員限制
memberName === 'XXXX' → 拋出錯誤（XXXX 會員不開放）

// 通路限制
possale.channel 必須 === 'STORE'

// 數量驗證
items[].amount <= possale.items[].amount - 退貨數量

// 自動建立調撥單
ctx.query.insert('transferorder', transferOrderBody)
```

### posdeposit — 寄貨單 Policy

```typescript
// 銷售單驗證
saleName 必須存在
possale.paidAt 必須存在

// 互斥驗證
同一 saleName 不能同時有 posdeposit 和 poscustomerorder

// 限制
memberName !== 'XXXX'
possale.channel !== '員購'

// 數量驗證
items[].amount <= possale.items[].amount - 已退貨數量 - 已寄貨數量

// 新增時自動設定
status = '新建'
items[].withdrawAmount = 0
```

### expenditure — 支出單 Policy（expenditurepolicy）

```typescript
// 新建時財務審核狀態
新建時 reviewStatus 必須 === '未審核'

// 調整項限制
type IN ['A006', 'A007', 'A039'] → 才能填負數金額

// 差旅限制
type IN ['B001', 'B002', 'B003', 'B004', 'B005'] → 特定欄位格式驗證
```

---

## BeforeInsert/BeforeUpdate 重點摘要

### possale — BeforeInsert

```typescript
// 計算稅額（每次新增自動計算）
record.totalTax = record.total - Math.round(record.total / 1.05)
```

### poscoupon — BeforeInsert

```typescript
// 標記料件集合需要更新
record.itemCollectionStatus = '待更新表頭與料件'
```

### poscoupon — BeforeUpdate

```typescript
// 若關鍵欄位變動，更新 itemCollectionStatus
const keyFields = ['startAt', 'endAt', 'filters', 'storeGroups', ...]
if (anyKeyFieldChanged) {
  record.itemCollectionStatus = '待更新表頭與料件'
}
```

### poseguino — BeforeInsert/BeforeUpdate

```typescript
// 驗證發票號碼格式
for (const interval of record.numberInterval) {
  if (!/^[A-Z]{2}[0-9]{8}$/.test(interval.startNo)) throw new Error(...)
  if (!/^[A-Z]{2}[0-9]{8}$/.test(interval.endNo)) throw new Error(...)
  if (interval.startNo >= interval.endNo) throw new Error(...)
}
```

### posdeposit — BatchBeforeInsert

```typescript
// 批次新增時驗證料件不重複
const allItemNames = records.flatMap(r => r.items.map(i => i.itemName))
if (hasDuplicates(allItemNames)) throw new Error('料件重複')
```

### posreport — BeforeInsert

```typescript
// 新增日結單時關聯機台
await ctx.query.update('posconfig', {
  filter: { name: record.configName },
  body: { reportName: record.name },
  userContent: { skipConfigUpdate: true }
})
```

---

## 重要的防迴圈 userContent 旗標

| 旗標 | 使用場景 | 說明 |
|-----|---------|------|
| `skipReturnUpdate` | posreturn policy → possale update | 退貨單寫入銷售單時防止 possale policy 再次觸發退貨邏輯 |
| `skipConfigUpdate` | posreport beforeInsert → posconfig update | 日結單建立時更新機台，防止機台 policy 再次觸發 |
| `skipExchangeUpdate` | possale → possale（換購） | 換購雙向更新時防止無限迴圈 |
| `TransferorderUserContent.skipBeforeUpdate` | customerOrder policy → transferorder | 建立調撥單時防止觸發調撥單的 beforeUpdate |
