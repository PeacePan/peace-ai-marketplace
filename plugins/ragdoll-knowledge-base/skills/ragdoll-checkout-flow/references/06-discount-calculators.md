# 10 個折扣計算器管道

**核心函數**：`calculateCheckoutDiscount()`，位於 `next/lib/stores/checkout/sale/use-sale.ts`

折扣計算採用**迭代模式（Iterative Pattern）**：每個計算器接收前一個計算器的輸出作為輸入，依序疊加折扣效果，最終結果存入 `sale.iteratedDiscount`。

---

## 迭代資料結構

定義於 `next/lib/stores/checkout/discount/type.ts`：

```typescript
interface CheckoutDiscountIterate {
  before: CheckoutDiscountState  // 傳入時的狀態（前一個計算器的 after）
  after: CheckoutDiscountState   // 計算完成後的狀態（傳給下一個計算器）
}

interface CheckoutDiscountState {
  body: SaleBody                     // 銷售單表頭（通路代號、會員名稱等）
  items: CheckoutDiscountSaleItem[]  // 商品明細列表
  promotions: DiscountPromotion[]    // 已套用的促銷紀錄清單
}
```

`promotions` 陣列中每筆紀錄的 `type` 欄位標記來源，完整型別詳見 [07-data-structures.md](./07-data-structures.md)。

---

## 計算器執行順序

```
初始輸入：
  items → 來自 useItems（normalItems + adjustPriceItems）
  body  → 來自會員資訊與門市設定
  promotions → []（空陣列，尚無促銷）

          │
          ▼
┌─────────────────────────┐
│  1. posAddonCalculator  │
│  Store: useAddons       │
└─────────────────────────┘
          │
          ▼
┌──────────────────────────┐
│  2. posFreebieCalculator │
│  Store: useFreebies      │
└──────────────────────────┘
          │
          ▼
┌────────────────────────────────────┐
│  3. petParkCouponItemsCalculator   │
│  Store: useMemberCoupon            │
└────────────────────────────────────┘
          │
          ▼
┌────────────────────────────┐
│  4. pointPromotionCalculator│
│  Store: usePointPromotion  │
└────────────────────────────┘
          │
          ▼
┌──────────────────────────────────────────┐
│  5. petParkCouponPromotionCalculator     │
│  Store: useMemberCoupon                  │
└──────────────────────────────────────────┘
          │
          ▼
┌──────────────────────────────┐
│  6. posPromotionCalculator   │  ← 最複雜，呼叫 Electron 端演算法
│  Store: usePosPromotion      │
└──────────────────────────────┘
          │
          ▼
┌────────────────────────────────────────────┐
│  7. petParkCouponDiscountCalculator        │
│  Store: useMemberCoupon                    │
└────────────────────────────────────────────┘
          │
          ▼
┌──────────────────────────────┐
│  8. orderDiscountCalculator  │
│  Store: useOrderDiscount     │
└──────────────────────────────┘
          │
          ▼
┌────────────────────────────┐
│  9. posCouponCalculator    │
│  Store: usePosCoupon       │
└────────────────────────────┘
          │
          ▼
┌──────────────────────────────┐
│  10. memberPointsCalculator  │
│  Store: useMemberPoints      │
└──────────────────────────────┘
          │
          ▼
最終輸出存入 sale.iteratedDiscount
```

---

## 各計算器詳細說明

### 計算器 1：`posAddonCalculator`

**職責**：將已選擇的加購商品加入計算流程。

**輸入**：`useAddons.addonItems`（`FreebieCheckoutItem[]`）

**邏輯**：
- 將每個加購商品轉換為 `saleItemType = FREEBIE_ADDON` 的 `CheckoutDiscountSaleItem`
- 使用實際售價（加購商品是有售價的，顧客需支付）
- 附加：關聯的活動名稱（`promotionName`），用於發票顯示 `[加]` 前綴

**輸出**：`items` 陣列新增加購行，`promotions` 不變

---

### 計算器 2：`posFreebieCalculator`

**職責**：將已選擇的贈品加入計算流程。

**輸入**：`useFreebies.freebieItems`（`FreebieCheckoutItem[]`）

**邏輯**：
- 將每個贈品轉換為 `saleItemType = FREEBIE_GIVE` 的 `CheckoutDiscountSaleItem`
- `price` 強制設為 `0`（贈品不收費）
- 本數量不計入訂單小計

**輸出**：`items` 陣列新增贈品行（0 元）

---

### 計算器 3：`petParkCouponItemsCalculator`

**職責**：處理會員優惠券觸發的特定商品（如贈品券兌換商品、免費服務料件）。

**輸入**：`useMemberCoupon` 中已選擇且驗證成功的 PetPark 優惠券

**邏輯**：
- 分析優惠券類型，若為「商品兌換型」則將對應商品加入 `items`
- 商品的 `saleItemType` 標記為 `PET_PARK_COUPON`

**輸出**：`items` 陣列新增優惠券觸發的商品行

---

### 計算器 4：`pointPromotionCalculator`

**職責**：將點加金兌換商品加入計算流程。

**輸入**：`usePointPromotion.pointItems`（`PointPromotionStateItem[]`）

**邏輯**：
- 每筆點加金商品分為「點」和「金」兩部分：
  - `price` = 「金」的部分（顧客支付的金額）
  - `pointsPromotionName` = 「點」的部分（記錄活動名稱，供後端銷帳）
- `saleItemType` 標記為 `POINT_PLUS_MONEY`

**輸出**：`items` 陣列新增點加金行，點數部分以 metadata 形式保留

---

### 計算器 5：`petParkCouponPromotionCalculator`

**職責**：套用 PetPark 優惠券的「商品折扣型」效果（針對特定商品的百分比或直減折扣）。

**輸入**：`useMemberCoupon` 中已驗證的折扣型優惠券

**邏輯**：
- 根據優惠券設定，找出 `items` 中符合條件的商品
- 套用折扣（百分比 `percentage` 或直減 `amount`）
- 更新這些商品的 `price`

**輸出**：
- 符合條件商品的 `price` 更新
- `promotions` 新增 `PET_PARK_COUPON_PROMOTION` 類型紀錄

---

### 計算器 6：`posPromotionCalculator`（最複雜）

**職責**：核心商品促銷計算，套用 POS 後台設定的促銷活動（滿件、滿額、ABC 組合等）。

**輸入**：當前所有 `items`、會員身分資訊

**邏輯步驟**：

```
1. 篩選候選促銷（validPromotions）：
   - 來源：candidatePosPromotions（由 refreshCandidatePosPromotions subscription 維護）
   - 篩選條件：isCandidatePosPromotion（活動期間、適用通路、會員身分、商品篩選條件）
   - 排序：level ASC → startAt DESC → createdAt DESC
   - 合併口袋促銷（pocketPosPromotions）→ 建立 combinedPromotionsMap

2. 區分商品類型（三分支 if/else 判定，依序優先）：

   a. 員工價商品（employeePrice）：
      條件：itemInfo.employeePromotionName 存在
            + itemInfo.employeePrice 為 number
            + 當前會員為員工（isUser）
      處理：
        - 從 combinedPromotionsMap 查找促銷
        - 若找不到 → 從 candidatePosPromotions 候補查找
          （因 refreshCandidatePosPromotions 會繞過 isCandidatePosPromotion 直接加入）
        - 若候補也找不到 → fallback 到 candidateItems 參與最佳促銷計算
        - 找到促銷 → 以 (basePrice - employeePrice) × amount 計算折扣
        - 透過 addItemsToPromotionMap 加入 dealPosPromotionItems

   b. 指定商品單價（promotionPrice）：
      條件：itemInfo.promotionName 存在
            + itemInfo.promotionPrice 為 number
      處理：
        - 從 combinedPromotionsMap 查找促銷
        - 若找不到 → 從 candidatePosPromotions 候補查找（同上）
        - 若候補也找不到 → fallback 到 candidateItems 參與最佳促銷計算
        - 找到促銷 → 以 (basePrice - promotionPrice) × amount 計算折扣
        - 透過 addItemsToPromotionMap 加入 dealPosPromotionItems

   c. 一般商品：
      不符合 a/b → 加入 candidateItems 交給最佳促銷演算法

3. 呼叫 ragdollAPI.calcBestPosPromotion（Electron 端 Worker 執行）：
   - 輸入：candidateItems（排除步驟 2a/2b 的商品）+ 候選促銷清單
   - 演算法尋找讓顧客獲得最大折扣的促銷組合
   - 輸出：bestAnswer 含分桶結果（有促銷的桶 + 無促銷的桶）

4. 支援的促銷類型：
   - 滿件折扣（OFF）：
     * 連續計算模式：根據商品總件數持續匹配門檻（如買 3 件折 9 折，買 6 件折 8 折）
     * 非連續計算：依門檻由大到小逐一比對，找最高可套用的門檻
   - 滿額折扣：達到金額門檻給予整批折扣
   - ABC 促銷：A 類商品 + B 類商品購買時折扣 C 類商品
   - PRICE_BY_ITEM：指定料件售價（由步驟 2b 處理，不進入演算法）

5. 根據 discountBase 決定計算基準：
   - MEMBER：以會員價（memberPrice）為計算基礎
   - LABEL：以定價（labelPrice）為計算基礎
```

**candidatePosPromotions 候補機制**：

料件的 `promotionName`（會員指定單價）和 `employeePromotionName`（員購指定單價）是直接寫在料件主檔上的欄位。`refreshCandidatePosPromotions` 會將這些促銷**繞過** `isCandidatePosPromotion` 篩選直接加入 `candidatePosPromotions`，但 `posPromotionCalculator` 建立 `validPromotions` 時會用 `isCandidatePosPromotion` **重新過濾**，可能將這些促銷濾除。因此計算器在查找促銷時採用兩階段查找：先查 `combinedPromotionsMap`，找不到再從 `candidatePosPromotions` 候補查找。

**輸出**：
- `items` 中受促銷影響的商品 `price` 更新（含員工價/指定單價/最佳促銷組合）
- `promotions` 新增 `PROMOTION` 類型紀錄（含活動名稱、折扣金額）

---

### 計算器 7：`petParkCouponDiscountCalculator`

**職責**：套用 PetPark 優惠券的「整單折扣型」效果。

**輸入**：`useMemberCoupon` 中已驗證的整單折扣型優惠券

**邏輯**：整張訂單均一折扣（不針對特定商品），計算折抵金額加入 promotions。

**輸出**：`promotions` 新增 `PET_PARK_COUPON_DISCOUNT` 類型紀錄

---

### 計算器 8：`orderDiscountCalculator`

**職責**：套用系統設定的整單滿額折扣。

**輸入**：計算器 1-7 後的商品小計

**邏輯步驟**：

```
1. 以步驟 1-7 後的商品小計為計算基準

2. 取得所有有效的整單折扣活動，檢查各活動門檻（min）

3. 計算方式（依活動設定）：
   a. percentage 模式：
      折扣金額 = 小計 × (1 - percentage)
   b. 級距金額模式：
      每滿 N 元折 M 元（含上限 max）

4. 多活動競爭規則：
   選折抵金額最高且門檻最低的「最佳解」

5. 排除規則：
   若訂單中商品全為點加金商品或贈品券商品，
   通常不套用整單折扣（依後端活動設定決定）
```

**輸出**：`promotions` 新增 `ORDER_DISCOUNT` 類型紀錄

---

### 計算器 9：`posCouponCalculator`

**職責**：套用銷售員手動刷入的折扣代碼。

**輸入**：`usePosCoupon` 中已驗證的折扣代碼資訊

**邏輯**：
- 驗證折扣代碼的有效性（格式、期間、使用次數限制）
- 計算折扣：百分比（`percentage`）或直減金額（`amount`）

**輸出**：`promotions` 新增 `COUPON_CODE` 類型紀錄

---

### 計算器 10：`memberPointsCalculator`

**職責**：套用會員點數折抵現金。

**輸入**：
- `useMemberPoints` 的期望折抵金額（`desiredDiscount`）
- `PointDiscount` 設定（匯率 `rate`、上限百分比 `maxPercentage`）

**邏輯步驟**：

```
1. 依匯率計算所需點數：
   所需點數 = desiredDiscount × rate
   例：折抵 $100，匯率 10 點/元 → 需 1,000 點

2. 驗證上限 1：
   if (desiredDiscount > saleTotal × maxPercentage)
     → 將折抵金額縮減至上限

3. 驗證上限 2：
   if (desiredDiscount > remainingTotal)
     → 折抵金額取 remainingTotal（避免產生負數應付金額）

4. 最終折抵金額 = min(驗證後金額, 會員可用點數換算金額)
```

**輸出**：`promotions` 新增 `DISCOUNT_BY_POINTS` 類型紀錄

---

## 金額計算彙總公式

```
saleTotal = 商品加購點加金的原始總額
            - 計算器 1-6 產生的商品層級促銷折扣

total     = saleTotal
            - 計算器 7 的 PetPark 整單優惠券折扣
            - 計算器 8 的整單滿額折扣
            - 計算器 9 的折扣碼折扣
            - 計算器 10 的點數折抵

discount  = 原始商品總額 - total
            （記錄於 offline_sale.body.discount）
```

> **依賴原則**：整單折扣（8-10）以商品促銷後的小計（`saleTotal`）為基準，確保折扣不會重複疊加在原始定價上。
