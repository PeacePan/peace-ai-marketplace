# 快照表格

## 快照機制概述

POS 系統的促銷活動（pospromotion/poscoupon/posorderdiscount）是可以修改的，但已完成的銷售單必須記錄結帳當時的活動設定（否則退貨時無法重現原始折扣）。因此設計了「快照機制」：

**快照建立時機**：每次修改促銷活動時，在 Policy 中自動建立一份快照。

**快照使用時機**：銷售單 Policy 中呼叫 `Pos.checkSnapshot()` 驗證銷售單記錄的活動版本是否與快照一致。

```
促銷活動修改 → policy.ts 呼叫 Pos.insertSnapshot() → 建立快照記錄
                                                          ↓
銷售完成時 → possale policy 呼叫 Pos.checkSnapshot() → 比對快照
```

---

## poscouponsnapshot — 折扣碼快照（v17）

### 基本資訊

- **表格代碼**：`POS_COUPON_SNAPSHOT`
- **dependencyTables**：`poscoupon`（繼承欄位）

### 結構特性

```typescript
// 繼承 poscoupon 的欄位，但排除以下欄位：
excludeFields: ['name', 'code', 'department', 'activityType']

// 新增快照專屬欄位：
name        // 快照編號：${poscoupon.name}_${poscoupon._updatedAt}
code        // 折扣碼代碼（從原表複製）
department  // 部門別（從原表複製）
activityType // 活動類型（從原表複製）
```

### 快照編號規則

快照編號格式：`{原折扣碼名稱}_{更新時間戳}`

例如：`CC000001-S_2024031512345678`

這使得同一折扣碼的不同版本可以被唯一識別。

---

## pospromotionsnapshot — 促銷快照（v26）

### 基本資訊

- **表格代碼**：`POS_PROMOTION_SNAPSHOT`
- **dependencyTables**：`pospromotion`（繼承欄位）

### 結構特性

```typescript
// 繼承 pospromotion 的欄位，排除部分欄位後新增：
name         // 快照編號：${pospromotion.name}_${pospromotion._updatedAt}
// 加上 priceCardColor（貨價卡顏色）等快照需要的額外欄位
```

### 用途

銷售單中每個料件可以關聯到促銷快照（`items[].promotionName REF → pospromotionsnapshot`），確保退貨時能重現當時的促銷設定。

---

## posorderdiscountsnapshot — 整單折扣快照（v18）

### 基本資訊

- **表格代碼**：`POS_ORDER_DISCOUNT_SNAPSHOT`
- **dependencyTables**：`posorderdiscount`（繼承欄位）

### 結構特性

```typescript
// 繼承 posorderdiscount 的欄位，排除部分欄位後新增：
name  // 快照編號：${posorderdiscount.name}_${posorderdiscount._updatedAt}
```

---

## 快照操作 API

### 建立快照（在 Policy 中呼叫）

```typescript
// poscoupon/policy.ts, pospromotion/policy.ts, posorderdiscount/policy.ts 中
await Pos.insertSnapshot(ctx, 'poscoupon', currentRecord)
await Pos.insertSnapshot(ctx, 'pospromotion', currentRecord)
await Pos.insertSnapshot(ctx, 'posorderdiscount', currentRecord)
```

### 比對快照（在 possale Policy 中呼叫）

```typescript
// possale/policy.ts 中
await Pos.checkSnapshot(ctx, 'pospromotion', promotionNames)
await Pos.checkSnapshot(ctx, 'poscoupon', couponNames)
await Pos.checkSnapshot(ctx, 'posorderdiscount', orderDiscountNames)
```

### 快照比對失敗情境

當銷售單記錄的活動版本（快照編號）與目前最新快照不一致時，`checkSnapshot` 會拋出錯誤，表示：
1. 促銷活動在顧客結帳過程中被修改了
2. 前端記錄的活動設定已過時

此時 POS 前端需重新載入活動設定，重新計算折扣。

---

## 快照表的開發注意事項

1. **快照表不應直接修改**：快照是唯讀的歷史記錄，不應手動修改或刪除。

2. **dependencyTables 表示繼承**：修改 pospromotion 的欄位時，記得考慮 pospromotionsnapshot 是否也需要同步調整（新增的欄位可能需要加入快照）。

3. **version 同步遞增**：poscoupon 的 version 遞增時，poscouponsnapshot 的 version 通常也會同步遞增（因為 dependencyTables 機制會重新產生欄位）。

4. **查詢快照用途**：查詢某筆銷售的當時促銷設定，可透過銷售單的 `items[].promotionName` 查到 pospromotionsnapshot，即可重現當時的促銷條件。
