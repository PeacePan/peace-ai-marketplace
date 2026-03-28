# 促銷活動表格

## pospromotion — 商品促銷（v80）

### 基本資訊

- **表格代碼**：`POS_PROMOTION`
- **自動編碼**：`AA[6碼] + {channelCode後綴}`（例：AA000001-S）
- **jobQueues**：`'1,1,1,1,1'`（5個 queue）
- **dependencyTables**：`pospromotionsnapshot`
- **欄位分組**：BASIC / SETTING / EMARSYS / SYSTEM

### 促銷類型（PosPromotionType）

| 類型值 | 說明 |
|-------|------|
| ITEM_PRICE | 指定商品價錢（可為多商品組合） |
| GROUP_ITEM_SAME_PRICE | 群組商品同單價 |
| COMBINATION | 組合價（多商品一起購買） |
| QUANTITY_DISCOUNT | 滿件折扣（達到件數給折扣） |
| SECOND_ITEM_DISCOUNT | 第二件折扣 |

### 核心欄位

```typescript
// BASIC
name            // 促銷編號（自動）
displayName     // 促銷名稱
startAt / endAt // 活動時間
type            // 促銷類型（ENUM: PosPromotionType）
target          // 適用對象（ENUM: PosPromotionTarget - 全部/會員/非會員）
desc            // 說明
department      // 部門別
activityType    // 活動類型
priceCardPrintName  // 貨價卡名稱
priceCardColor      // 貨價卡顏色（ENUM）

// SETTING
min / max              // 最小/最大數量
continuous             // 是否連續計算
memberMax              // 會員使用上限
allStoreMax            // 全店上限
storeUsedAmount        // 已用門市上限（READ_NULLABLE）
percentage             // 折數（1-100，null=不打折）
discountBase           // 折扣基底
directPrice            // 指定價格
discount               // 折抵金額
level                  // 折扣層級
channelCode            // 通路代碼
itemPromotionPriceName // 料件促銷售價名稱

// EMARSYS
label  // Emarsys 行銷標籤

// SYSTEM
itemCollectionStatus  // 料件集合同步狀態
itemCollectionError   // 料件集合同步錯誤
```

### 表身（LineSchema）

```typescript
items: {
  itemName: REF → ITEM,
  price: 指定金額（ITEM_PRICE 類型使用）
}

filters: {
  // 料件篩選器（決定活動適用哪些料件）
  filterName: 篩選器名稱（ENUM: PosItemFilterName）,
  operator: 運算子（ENUM: PosItemFilterOperator）,
  value: 篩選值
}

zones: {
  filterName: 篩選器名稱,
  min: 最小值（輔助 filters 的數量限制）
}

storeGroups: {
  groupName: 門市群組（REF → posstoregroup）,
  effect: 效果（ENUM: PosFilterEffect - INCLUDE/EXCLUDE）,
  memo: 備註
}

weekly: { day: 1-7 }    // 適用星期幾
monthly: { date: 1-31 } // 適用每月幾號

effects: {
  // 滿件折扣的層級效果
  min: 最小件數,
  percentage: 折數,
  discountBase: 折扣基底,
  directPrice: 指定價格,
  discount: 折抵金額
}
```

### 重要腳本

**Policy**：
- 促銷類型限制（不同類型的欄位互斥驗證）
- 打折/指定價格/折抵金額三選一
- 表身篩選器與料件售價相依性
- `effects` 表身層級與連續計算設定
- Emarsys 標籤檢查
- **建立/修改時自動建立快照**：`Pos.insertSnapshot(ctx, 'pospromotion', record)`

**Cron**：
- `門市上限刷新`（HALF_HOURLY）
- `活動料件集合檔刷新`（HALF_HOURLY）
- `同步Foodomo售價`（定時）

---

## poscoupon — 折扣碼（v30）

### 基本資訊

- **表格代碼**：`POS_COUPON`
- **自動編碼**：`CC[6碼] + {channelCode後綴}`；`code: S[6碼] + {channelCode後綴}`
- **jobQueues**：`'1,1,1'`（3個 queue）
- **dependencyTables**：`poscouponsnapshot`

### 折扣碼類型（PosCouponType）

| 類型值 | 說明 |
|-------|------|
| AMOUNT | 指定金額折扣 |
| PERCENTAGE | 折數折扣 |
| ITEM | 商品滿額折扣 |

### 發行方式（issueType）

| 發行方式 | 說明 |
|---------|------|
| 公開發行 | 所有人可輸入 code 使用 |
| 批次發行 | 需透過 poscouponcodegenerator 產生序號才能使用 |

### 核心欄位

```typescript
name         // 折扣碼編號（自動）
displayName  // 折扣碼名稱
code         // 折扣碼（S+[6碼]+後綴，公開發行使用）
startAt / endAt  // 活動時間
type         // 折扣類型（ENUM: PosCouponType）
issueType    // 發行方式
limit        // 使用次數上限
discount     // 折抵金額
min          // 最低消費門檻
max          // 最高折抵上限
department   // 部門別
activityType // 活動類型
channelCode  // 通路代碼
label        // Emarsys 行銷標籤
itemCollectionStatus / itemCollectionError  // 料件集合同步
```

### 表身（LineSchema）

```typescript
filters: { /* 同 pospromotion */ }
storeGroups: { /* 同 pospromotion */ }
weekly: { day: 1-7 }
monthly: { date: 1-31 }

exclusiveCoupon: {
  // 與此折扣碼互斥的其他折扣碼
  couponName: REF → poscoupon
}

// 外部表（EXTERNAL）
posCouponCodeGenerators: {
  // 讀取 POS_COUPON_CODE_GENERATOR 的批次產生記錄
  lineType: EXTERNAL,
  externalTableName: POS_COUPON_CODE_GENERATOR
}
```

### 重要腳本

**Policy**：
- ITEM 類型必須設定料件篩選器
- 不可重複設定周/月適用日
- Emarsys 標籤驗證
- **建立/修改時自動建立快照**

**BeforeUpdate**：
- 若關鍵欄位變動，觸發更新 `itemCollectionStatus`

**Cron（HALF_HOURLY）**：
- `活動料件集合檔刷新`

---

## poscouponcode — 折扣序號（v4）

### 基本資訊

- **表格代碼**：`POS_COUPON_CODE`
- **自動編碼**：`autoKeyNonSerialId`（非流水號，唯一隨機 ID）

### 核心欄位

```typescript
name            // 序號編號（自動，非流水號）
generatorName   // 批次產生器（REF → poscouponcodegenerator）
couponName      // 所屬折扣碼（REF → poscoupon）
redemptionCode  // 兌換碼（INSERT 時可讀寫，即批次產生的唯一兌換碼）
saleName        // 使用銷售單（REF → possale，使用後填入）
```

### 函式

```typescript
// 使用序號（舊方式，execute）
使用序號(ctx, { couponCodeName })

// 使用序號V2（新方式，call，透過兌換碼查詢）
使用序號V2(ctx, { redemptionCode })
```

---

## poscouponcodegenerator — 折扣碼批次產生（v1）

### 基本資訊

- **表格代碼**：`POS_COUPON_CODE_GENERATOR`
- **自動編碼**：`{couponName前綴} + initialNumber: 1`
- **jobQueues**：`'1,1,1,1,1'`（5個 queue）

### 核心欄位

```typescript
name               // 批次編號（自動）
couponName         // 折扣碼（INSERT_REF → poscoupon，draftable）
targetGenerateCount  // 目標產生組數（INSERT_INT，min=1）
generatedCount       // 已產生組數（READ，scriptReadWrite: UPDATE，defaultValue: '0'）
cronStatus           // 排程狀態（未執行/執行中/已完成/錯誤）
cronError            // 排程錯誤
```

### 重要限制

**Policy**：
- 折扣碼發行方式必須為「批次發行」才能建立產生器

**函式**：
- `產生序號`（isPublic: true，onJobRetry: 1）：逐一建立 `poscouponcode` 記錄

---

## posorderdiscount — 整單折扣（v32）

### 基本資訊

- **表格代碼**：`POS_ORDER_DISCOUNT`
- **自動編碼**：`BB[6碼] + {channelCode後綴}`
- **jobQueues**：`'1,1,1,1,1'`（5個 queue）
- **dependencyTables**：`posorderdiscountsnapshot`
- **欄位分組**：BASIC / SETTING / EMARSYS / SYSTEM

### 核心欄位

```typescript
name         // 編號（自動）
displayName  // 名稱
target       // 適用對象（ENUM: PosPromotionTarget）
desc         // 說明
department   // 部門別
activityType // 活動類型
min / max    // 最低消費/最高折抵
memberMax    // 會員使用上限
allStoreMax  // 全店上限
storeUsedAmount  // 已用門市上限（READ）
startAt / endAt  // 活動時間
percentage   // 折數
discount     // 折抵金額
maxDiscountAmount  // 最高折扣金額
channelCode  // 通路代碼
label        // Emarsys 標籤
itemCollectionStatus / itemCollectionError
```

### 重要腳本

**Policy**：
- 打折與折抵金額互斥
- 不可重複周/月適用日
- **建立/修改時自動建立快照**

**Cron（2個）**：
- `活動料件集合檔刷新`
- `門市上限使用數量刷新`

---

## posfreebie — 加贈活動（v25）

### 基本資訊

- **表格代碼**：`POS_FREEBIE`
- **自動編碼**：`EE[6碼] + {channelCode後綴}`
- **jobQueues**：`'1,1,1,1,1'`（5個 queue）

### 加贈類型（PosFreebieType）

| 類型 | 說明 |
|------|------|
| ADDON | 加購（達到條件可加購商品） |
| FREEBIE | 贈品（達到條件免費贈品） |
| ITEM_AMOUNT | 商品滿額贈（指定料件達到金額） |
| ITEM_QUANTITY | 商品滿件贈（指定料件達到件數） |

### 核心欄位

```typescript
name         // 活動編號（自動）
displayName  // 活動名稱
startAt / endAt  // 活動時間
type         // 加贈類型（ENUM: PosFreebieType）
target       // 適用對象（ENUM: PosPromotionTarget）
min / max    // 最小/最大滿足條件
giveAmount   // 贈送數量
addonPrice   // 加購金額（ADDON 類型使用）
memberLimitSet  // 會員限量
storeLimitSet   // 門市限量
storeUsedAmount // 已用門市限量（READ）
department   // 部門別
activityType // 活動類型
label        // Emarsys 標籤
```

### 表身（LineSchema）

```typescript
sourceItemFilters: { /* 滿額/滿件的來源料件篩選器 */ }
targetItemFilters: { /* 贈品/加購的目標料件篩選器 */ }
storeGroups: { /* 門市群組設定 */ }
weekly: { day: 1-7 }
monthly: { date: 1-31 }

exclusiveFreebie: {
  // 與此活動互斥的其他加贈活動
  freebieName: REF → posfreebie
}
```

### 重要腳本

**Policy**：
- ITEM_AMOUNT/ITEM_QUANTITY 類型必須設定 `sourceItemFilters`
- 會員限量只有 target=會員 才能設定
- ADDON 類型必須填加購金額
- 不可重複周/月適用日

**Cron（HALF_HOURLY）**：
- `門市上限使用數量刷新`（isPublic: true）

---

## posvoucher — 截角禮券活動（v16）

### 基本資訊

- **表格代碼**：`POS_VOUCHER`
- **自動編碼**：`DD[6碼] + {channelCode後綴}`
- **jobQueues**：`'1'`（1個 queue）

### 核心欄位

```typescript
name         // 活動編號（自動）
displayName  // 活動名稱
startAt / endAt  // 活動時間
type         // 禮券類型
amount       // 禮券金額
memo         // 備註
channelCode  // 通路代碼
itemCollectionStatus / itemCollectionError
```

### 重要腳本

**Policy**：
- 不可重複周/月適用日

**Cron（DAILY，UTC 04:00）**：
- `活動料件集合檔刷新`

---

## positemcollection — 料件活動集合（v39）

### 基本資訊

- **表格代碼**：`POS_ITEM_COLLECTION`
- **自動編碼**：`{eventName前綴}YYMMDD[2碼]`
- **jobQueues**：`'1,1,1,1,1'`（5個 queue）

### 用途

料件活動集合是促銷活動的「料件快取」。每個活動（pospromotion/poscoupon/posorderdiscount/posfreebie/posvoucher）都有一個對應的 positemcollection，記錄活動目前涵蓋的所有料件。

### 核心欄位

```typescript
name             // 集合編號（自動）
eventSource      // 活動來源（ENUM: ItemCollectionSource）
eventName        // 活動編號
eventDisplayName // 活動名稱
eventStartAt / eventEndAt  // 活動時間
eventType        // 活動類型
eventTarget      // 適用對象
eventDescription // 說明
eventChannel     // 通路
priceCardColor   // 貨價卡顏色
department       // 部門別
activityType     // 活動類型
itemSyncStatus   // 料件同步狀態
itemSyncAt       // 料件同步時間
itemSyncError    // 料件同步錯誤
```

### 表身（READ 唯讀，scriptReadWrite: FULL）

```typescript
items: {
  itemName            // 料件編號
  itemDisplayName     // 料件名稱
  memo                // 備註
  itemBrandName       // 品牌
  itemBrandDisplayName  // 品牌名稱
  itemCategoryDisplayName  // 類別名稱
}
```

### 核心業務邏輯：syncItemsTodoJob（272行）

```typescript
// 料件同步流程
1. 從來源活動讀取篩選器設定（filters）
2. 呼叫 Procurement.findItemsForItemCollection() 查詢符合條件的料件
   - chunkSize: 30000（大批查詢）
3. 比對現有集合表身：
   - 新增：$push（符合但不存在）
   - 移除：$pull（不符合但存在）
   - 更新：$set（符合且存在但欄位不同）
4. 執行超時控制：executeFunctionWithTimeout(fn, 48秒)
5. 記錄各項執行耗時（TimeRecorder）
```

**Cron（3個）**：
- `syncItems`：定時同步料件
- `moveOldData`：搬移舊資料
- `posFreebieItemCollection`：加贈活動料件集合同步
