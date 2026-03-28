# 會員相關表格（POS 側）

> **注意**：本文件涵蓋 POS 側與會員直接相關的表格（pospet、pospetspecies、posprintablecoupon、positemcollection、posstoreshelf）。
> 會員主檔（posmember）及完整會員系統知識請參閱 SKILL `norwegianforest-knowledge-base:member-system-knowledge`。

---

## pospet — 寵物檔（v4）

### 基本資訊

- **表格代碼**：`POS_PET`
- **自動編碼**：`[流水號8碼]`（initialNumber: 1，autoKeyMinDigits: 8）

### 用途

記錄會員的寵物資料。寵物是 POS 系統中重要的業務維度，影響美容預約、點數贈送、優惠券發放等。一個會員可以有多隻寵物。

### 核心欄位

```typescript
name         // 寵物編號（自動）
displayName  // 寵物名稱
species      // 品種（REF → pospetspecies）
category     // 分類（ENUM: PosPetCategory - DOG/CAT/SMALL_PET/OTHER）
status       // 狀態（ENUM: PosPetStatus - 正常/封存）
gender       // 性別（ENUM: PosMemberGender - 公/母/未知）
startDate    // 出生日期
memberName   // 所屬會員（REF → posmember）
userName     // 建立人員
memo         // 備註
monthCardMemo // 月卡備註
```

### 表身（LineSchema）

```typescript
tags: {
  name  // 標籤名稱（isLineKey: true，自訂標籤）
}
```

### 重要腳本

**BeforeInsert**：
- 自動設定初始狀態、預設值

**Policy**：
- 會員存在性驗證
- 品種與分類一致性驗證（例：品種為貓科，分類必須為 CAT）
- 封存的寵物不能再次操作

---

## pospetspecies — 寵物品種（無版本號）

### 基本資訊

- **表格代碼**：`POS_PET_SPECIES`
- **自動編碼**：無（手動輸入品種名稱）

### 用途

寵物品種的參考資料表。由管理員維護，前端查詢供選擇。

### 核心欄位

```typescript
name      // 品種名稱（KEY_FIELD，手動輸入）
category  // 分類（INSERT_ENUM: PosPetCategory）
memo      // 備註
```

---

## posprintablecoupon — 小白單優惠券（v14）

### 基本資訊

- **表格代碼**：`POS_PRINTABLE_COUPON`
- **自動編碼**：`MK[6碼]`
- **欄位分組**：BASIC / CONDITION / FORMAT

### 用途

「小白單」是結帳後列印的紙本優惠券。當顧客完成結帳（possale finishCheckout）後，系統根據銷售內容篩選符合條件的小白單活動，印出對應的優惠券給顧客。

### 篩選邏輯

結帳後列印小白單的篩選順序：
1. 活動時間在有效期內（startAt ≤ 現在 ≤ endAt）
2. 通路符合（printChannel）
3. 門市群組符合（storeGroups）
4. 料件篩選器符合（filters）
5. 消費金額達到門檻（saleTotal）
6. 優先序（priority）排序，取最高優先序活動

### 核心欄位

```typescript
// BASIC 基本資訊
name               // 優惠券編號（自動）
displayName        // 優惠券名稱
memo               // 備註
cooperationPartner // 合作廠商
priority           // 優先序（ENUM: PosPrintableCouponPriority）
startAt / endAt    // 活動時間
printChannel       // 列印通路（ENUM: PosPrintableCouponPrintChannel）

// CONDITION 條件設定
couponEventName      // 優惠券活動（REF → petparkcouponevent，可選）
printLimit           // 每次結帳最多印幾張
needCollect          // 是否需要集點
saleTotal            // 最低消費金額門檻
onlyFirstConsumption // 僅限首次消費
targetConsumer       // 目標顧客（ENUM: PosPrintableCouponTargetConsumer）

// 期間累積條件
accumulationStartAt / accumulationEndAt  // 累積消費期間
accumulationSaleTotal                    // 累積消費門檻

// FORMAT 格式設定
couponTop      // 優惠券頂部文字
barcodeContent // 條碼內容
barcodeType    // 條碼類型（ENUM: PosBarcodeType - CODE128/QR_CODE/CODE39/etc.）
title1/2/3     // 三行標題文字（各有：isBold、fontSize、align 設定）
expirationStartAt / expirationEndAt  // 優惠券有效期
```

### 表身（LineSchema）

```typescript
filters: {
  // 料件篩選器（決定哪些商品可觸發此優惠券）
  filterName: ENUM（PosItemFilterName）
  operator: ENUM（PosItemFilterOperator）
  value: 篩選值
}

storeGroups: {
  groupName  // 門市群組（REF → posstoregroup）
  effect     // INCLUDE/EXCLUDE
  memo       // 備註
}

couponPrintMemo: {
  // 優惠券備註列（可選）
  content  // 備註內容
}
```

### 重要腳本

**Policy**：
- 料件篩選器格式驗證（`itemFilterPolicy`）
- 門市群組驗證（`storeGroupPolicy`）
- 優先序與通路一致性驗證

---

## posstoreshelf — 門市台帳（v5）

### 基本資訊

- **表格代碼**：`POS_STORE_SHELF`
- **自動編碼**：無（手動命名）

### 用途

記錄門市商品的陳列位置（台帳）。每個門市對應一個台帳記錄，包含多個台帳群組（shelfGroup）。

### 核心欄位

```typescript
name       // 門市台帳編號（手動）
storeName  // 門市（unique，REF → posstore，每門市只有一筆）
```

### 表身（LineSchema）

```typescript
shelfGroup: {
  minLength: 1,   // 至少 1 個群組
  shelfGroupName  // 台帳群組（REF → POS_SHELF_GROUP，isLineKey: true）
  noList          // 台帳清單（FUNCTION_FIELD，functionName: getNewestNoList）
}
```

### 重要腳本

**函式（3個）**：
1. `getNewestNoList`（CALL function）：取得最新台帳清單
2. `台帳生效通知`（isPublic: true）：通知門市台帳已生效
3. `下載台帳圖`（FunctionType.URL，isPublic: true）：下載台帳圖片

---

## 會員系統的 POS 觸發流程

### 結帳後的會員更新鏈

```
possale.finishCheckout()
    │
    ├─ TodoJob: posmember/完成結帳後更新會員資料
    │   ├─ 更新消費金額、消費次數、最後消費日
    │   ├─ 計算會員等級升降級
    │   └─ 標記 emSyncStatus = 待同步（觸發 Emarsys 同步）
    │
    ├─ TodoJob: posmember/更新會員品類標籤
    │   └─ 根據銷售料件更新品類消費標籤（同步到 Emarsys）
    │
    └─ 小白單：updatePrintableCoupon.ts
        └─ 根據 posprintablecoupon 篩選符合條件的優惠券
```

### 寵物與銷售的關聯

```
possale（銷售單）
    └── 美容銷售時記錄服務寵物（pospet.name）
         └── 影響：美容紀錄、寵物消費統計、月卡使用記錄
```

### 退貨後的會員處理

```
posreturn（退貨完成）
    └─ MemberRights.refundSalePoints()
        └─ 收回因此銷售單獲得的點數
```
