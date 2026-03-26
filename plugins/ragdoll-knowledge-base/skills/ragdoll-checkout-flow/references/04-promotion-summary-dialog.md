# Step 6：促銷結算對話框

---

**元件**：`next/app/checkout/components/promotion-summary-dialog/index.tsx`

此對話框是進入付款頁面前的最後確認畫面，分為左側商品概覽與右側優惠操作兩個區塊。使用者在右側進行的每次操作，都會重新觸發 `saleActions.calculateCheckoutDiscount()`，即時更新左側顯示的折扣結果。

---

## 左側面板：商品與促銷群組展示

**元件**：`promotion-summary-dialog/left-panel/promotion-item-list.tsx`

**顯示邏輯**：
- 依促銷活動將商品群組化，同一個促銷活動下的商品放在同一個區塊
- 分區塊顯示以下類型：
  - 一般商品（按套用的促銷活動分組）
  - 加購商品區塊（`FREEBIE_ADDON`）
  - 贈品區塊（`FREEBIE_GIVE`，標示 0 元）
  - 點加金商品區塊（`POINT_PLUS_MONEY`，顯示使用點數）
  - 整單折扣區塊（顯示各種整單折扣的折抵金額）

---

## 右側面板：三個優惠操作 Section

### 1. 會員優惠券（`member-coupon-section.tsx`）

**Store**：`next/lib/stores/checkout/discount/member-coupon/use-member-coupon.ts`

**UI 元素**：
- 文字輸入框（輸入券號）
- 「驗證」按鈕
- 驗證成功後顯示優惠券名稱與折抵金額

**執行流程**：
```
1. 使用者輸入券號 → 按下驗證

2. 呼叫驗證 API 確認：
   - 券號是否存在
   - 是否在有效期間內
   - 是否已被使用

3. 驗證成功 → 將券資訊存入 Store

4. 觸發 saleActions.calculateCheckoutDiscount()
   → 依序執行計算器，其中 petParkCouponDiscountCalculator
     會讀取已驗證的券，套用整單折扣

5. 更新左側面板顯示折後金額
```

---

### 2. 點數折抵（`member-points-section.tsx`）

**Store**：`next/lib/stores/checkout/discount/member-points/use-member-points.ts`

**UI 元素**：
- 數值輸入框（手動輸入折抵金額）
- 即時顯示：可用點數、可折抵金額上限

**三層狀態模型**

此元件使用三個互相關聯的狀態管理輸入與套用邏輯，除錯時必須同時追蹤三者的值：

| 狀態 | 型別 | 用途 | 關鍵行為 |
|------|------|------|---------|
| `editingValue` | `string \| null` | 編輯中的輸入值 | `null` = 非編輯狀態；`onFocus` 進入編輯、`handleBlur` 結束後設為 `null` |
| `lastAppliedAmountRef` | `Ref<number>` | 上次成功套用的金額 | 用於 `handleBlur` 判斷是否需要重新套用（值相同則跳過） |
| `totalDiscount` | `number`（store state） | store 中的實際折扣金額 | 由 `memberPointsCalculator` 設定，用於非編輯狀態下的顯示 |

**`handleBlur` 的守衛條件**：
1. `editingValue === null` → 直接 return（非編輯狀態，不處理）
2. `amount === lastAppliedAmountRef.current` → 直接 return（值未改變）

第 1 個守衛至關重要：確認按鈕的 `onMouseDown` 會觸發 `document.activeElement.blur()`，若輸入框先前已 blur 過（`editingValue` 已被設為 `null`），此時 `handleBlur` 再次觸發時必須直接返回，否則 `parseFloat(null || '0')` 得到 `0`，會錯誤地清除已套用的折扣。

**三重上限校準邏輯**

使用者輸入的折抵金額會自動套用以下三個上限，取**最小值**：

| 上限 | 來源 | 說明 |
|------|------|------|
| 上限 1 | `PointDiscount.orderDiscountLimit` | 後端設定的單筆訂單點數折抵上限百分比（例如最多折抵 20%） |
| 上限 2 | 當前應付小計 | 折抵金額不能超過訂單應付金額（不能產生負數應付） |
| 上限 3 | 會員剩餘可用點數換算金額 | 折抵金額不能超過會員現有點數可換算的最大金額 |

**執行流程**：
```
1. 使用者輸入折抵金額 → editingValue 更新

2. 使用者 blur（按 Enter 或點擊其他區域）→ handleBlur 觸發

3. 自動校準：取三重上限的最小值

4. 回收上次使用的點數（若有）

5. 呼叫 memberPointsActions.setDiscountAmount(finalAmount, subtotal)
   → 設定 store 的 usedPoints

6. 觸發 saleActions.calculateCheckoutDiscount()
   → memberPointsCalculator 讀取 usedPoints，
     產生 DISCOUNT_BY_POINTS 類型的 promotion 記錄

7. setEditingValue(null) → 結束編輯狀態
```

---

### 3. 折扣碼（`discount-code-section.tsx`）

**Store**：`next/lib/stores/checkout/discount/pos-coupon/use-pos-coupon.ts`

**UI 元素**：
- 文字輸入框（輸入折扣代碼）
- 「套用」按鈕
- 套用成功後顯示折扣代碼名稱與折抵金額

**執行流程**：
```
1. 使用者輸入折扣代碼 → 按下套用

2. 呼叫 API 驗證代碼有效性

3. 驗證成功 → 將折扣資訊（百分比或直減金額）存入 Store

4. 觸發 saleActions.calculateCheckoutDiscount()
   → posCouponCalculator 讀取折扣碼，
     計算並套用折扣

5. 更新左側面板與底部總額顯示
```

---

## 確認按鈕：前往付款

按下「確認結帳」後：

**對話框層**（`PromotionSummaryDialog`）：
```
1. handleConfirm 是 async 函式
2. await onConfirm()（等待 checkout/page.tsx 的確認流程完成）
3. 完成後才呼叫 onOpenChange(false) 關閉對話框
```

對話框必須延遲關閉的原因：確認按鈕的 `onMouseDown` 會觸發 `document.activeElement.blur()`，若 MemberPointsSection 的輸入框正在編輯中，blur 會啟動非同步的 `setDiscountAmount` + `calculateCheckoutDiscount`。對話框需保持掛載，確保這些非同步操作在元件存活期間完成。

**頁面層**（`handlePromotionSummaryConfirm` in `checkout/page.tsx`）：
```
1. 標記結算已確認（設定 isPromotionSummaryConfirmedRef，
   防止對話框關閉時觸發回滾）

2. connectionActions.checkInvoiceDevice()
   → 呼叫 IPC 確認發票機連線狀態

3. await saleActions.calculateCheckoutDiscount()
   → 重新執行完整折扣計算，確保所有 blur 觸發的狀態變更
     （點數折抵、優惠券、折扣碼等）都已反映至 iteratedDiscount

4a. 連線正常 → router.push('/summary')
4b. 連線異常 → 顯示警告 toast，阻擋跳轉
```
