# 配置支援表格

## posstoregroup — 門市群組（v2）

### 基本資訊

- **表格代碼**：`POS_STORE_GROUP`
- **自動編碼**：無（手動輸入群組名稱）

### 用途

門市群組是促銷活動（pospromotion/poscoupon/posorderdiscount/posfreebie/posvoucher）的「適用門市」設定基礎。每個促銷活動的 `storeGroups` 表身都引用 `posstoregroup`。

### 核心欄位

```typescript
name  // 群組名稱（手動輸入，KEY_FIELD）
memo  // 備註
```

### 表身（LineSchema）

```typescript
stores: {
  minLength: 1,   // 至少 1 個門市
  store: REF → posstore（isLineKey: true）,
  memo: 備註
}
```

---

## pospilot — 前導測試種子店（v6）

### 基本資訊

- **表格代碼**：`POS_PILOT`
- **自動編碼**：`PP[5碼]`

### 用途

管理 POS 功能的前導測試（Beta 測試）。當新功能需要先在少數門市測試時，建立 pospilot 記錄，指定測試門市與功能關鍵字。前端（Ragdoll）會讀取此配置決定是否開放特定功能。

### 核心欄位

```typescript
name        // 前導測試編號（自動）
version     // 版本（unique，前端依此查詢）
description // 描述
startAt     // 測試開始時間
endAt       // 測試結束時間
```

### 表身（LineSchema）

```typescript
stores: {
  storeName   // 測試門市（REF → posstore）
  configName  // 機台編號
}

features: {
  keyword  // 功能關鍵字（ENUM: PosPilotFeatureKeyword）
}
```

### 重要腳本

**Policy**：
- 測試時間範圍驗證
- 門市機台一致性驗證（configName 必須屬於 storeName）

---

## posshortcut — 鍵盤群組（v2）

### 基本資訊

- **表格代碼**：`POS_SHORTCUT`
- **自動編碼**：`[流水號8碼]`

### 用途

POS 收銀機的快捷鍵設定。每個鍵盤群組最多 32 個按鈕（buttons），每個按鈕可對應多個料件（items）。

### 核心欄位

```typescript
name         // 鍵盤群組編號（自動）
displayName  // 鍵盤群組名稱
```

### 表身（LineSchema）

```typescript
buttons: {
  orderNo     // 按鈕順序（1-32）
  displayName // 按鈕顯示名稱
}

items: {
  orderNo   // 對應按鈕順序
  itemName  // 料件（REF → ITEM）
  amount    // 數量
}

allowStoreGroups: {
  groupName  // 允許使用的門市群組（REF → posstoregroup）
}
```

### 重要腳本

**Policy**：
- orderNo 範圍驗證（1-32）
- allowStoreGroups 門市群組存在性驗證

---

## poseguino — 電子發票號碼管理（v5）

### 基本資訊

- **表格代碼**：`POS_EGUI_NO`
- **自動編碼**：`initialNumber: 1`
- **Library**：Utils, Pos

### 用途

管理電子發票號碼區段。每個收銀機（posconfig）的每個時期都有固定數量的發票號碼（startNo 到 endNo），系統透過 `offset` 追蹤目前用到哪個號碼。

### 核心欄位

```typescript
name       // 編號（自動）
storeName  // 門市（INSERT_REF → posstore）
applyYear  // 使用年份（民國年，min=110, max=199）
startMonth // 起始月份（1-12）
endMonth   // 結束月份（1-12）
```

### 表身（LineSchema）

```typescript
numberInterval: {
  configName  // 機台（INSERT_REF → posconfig）
  startNo     // 起始號碼（正則: ^[A-Z]{2}[0-9]{8}$，例：AA00000001）
  endNo       // 結束號碼
  offset      // 目前號碼偏移量（UPDATE_NULLABLE_INT，scriptReadWrite: UPDATE）
}
```

### 重要腳本

**BeforeInsert/BeforeUpdate Hook**：
- 驗證號碼格式（startNo/endNo 格式）
- 驗證 offset 不超過號碼區段範圍
- 防止號碼區段重疊

**Policy（poseguinopolicy）**：
- 門市與機台一致性驗證
- 年份與月份範圍驗證

### 電子發票取號流程

```typescript
// possale finishCheckout 中呼叫
Pos.pickEGUINo(record, ctx)
// → 找到對應 configName 的 numberInterval
// → offset + 1
// → 返回 startNo 加上 offset 計算得到的發票號碼
```

---

## expenditure — 支出單（v40）

### 基本資訊

- **表格代碼**：`EXPENDITURE`
- **自動編碼**：`initialNumber: 1`
- **enableApproval2**：`true`（啟用二階段審核）
- **欄位分組**：BASIC / FINANCE

### 用途

門市日常支出的申請與財務審核流程。支援多種支出類型（差旅、調整項、一般支出等），需要財務人員審核後才能報銷。

### 支出類型（ExpenditureType）

| 類型代碼 | 說明 |
|---------|------|
| A006 | 多存調整 |
| A007 | 少存調整 |
| A039 | 客訴調整 |
| B001-B005 | 差旅費相關 |
| 其他 | 一般支出 |

### 核心欄位

```typescript
// BASIC 基本資訊
name         // 支出單編號（自動）
storeName    // 門市（REF → posstore）
configName   // 機台（REF → posconfig）
reportName   // 日結單（REF → posreport）
userName     // 申請人
type         // 支出類型（ENUM: ExpenditureType）
intent       // 支出用途（ENUM: ExpenditureIntent）
memo         // 備註
amount       // 金額（min=0，調整項可為負數）
department   // 部門別

// FINANCE 財務資訊
tax              // 稅額（FUNCTION_FIELD: calculateTax，自動計算）
reviewStatus     // 審核狀態（ENUM: FinanceReviewStatus - 未審核/審核中/已通過/已拒絕）
reviewMemo       // 審核備註
reviewUser       // 審核人員
reviewAt         // 審核時間
reimbursementReportName  // 報銷單編號
```

### 重要腳本

**Policy（expenditurepolicy）**：
- 新建時財務審核狀態必須為「未審核」
- 調整項類型（A006/A007/A039）才能填負數金額
- 差旅類型（B001-B005）的特殊限制
- 審核狀態轉換限制

---

## posmallcategory — 商場類別（v3）

### 基本資訊

- **表格代碼**：`POS_MALL_CATEGORY`
- **自動編碼**：`PMC[6碼]`

### 用途

定義商場（Mall）的料件分類，作為 posmallitemsetting 和 posmallpromotionsetting 的類別依據。

### 核心欄位

```typescript
name         // 類別碼名稱（自動）
code         // 類別代碼（最多16位英數符號，正則: ^[A-Za-z0-9+]{1,16}$）
displayName  // 類別顯示名稱
```

---

## posmallitemsetting — 商場料件設定（v2）

### 基本資訊

- **表格代碼**：`POS_MALL_ITEM_SETTING`
- **自動編碼**：`PMIS[6碼] + {posMallCategoryName前綴}`

### 用途

設定商場特定時期對料件的篩選規則（包含或排除特定門市群組）。

### 核心欄位

```typescript
name                // 編號（autoKey，allowNull: true）
startAt / endAt     // 計算起始/結束時間
posMallCategoryName // 商場類別（REF → posmallcategory）
itemName            // 料件（REF → ITEM）
```

### 表身（LineSchema）

```typescript
storeGroups: {
  groupName  // 門市群組（REF → posstoregroup，isLineKey: true）
  effect     // 效果（ENUM: PosFilterEffect - INCLUDE/EXCLUDE）
  memo       // 備註
}
```

### 重要腳本

**Policy（3個）**：
1. `政策`：料件存在性驗證
2. `storeGroupPolicy('storeGroups')`：門市群組驗證
3. `noOverlapTimeIntervalPolicy('itemName', 'posmallitemsetting')`：同料件時間段不重疊

---

## posmallpromotionsetting — 商場促銷設定（v2）

### 基本資訊

- **表格代碼**：`POS_MALL_PROMOTION_SETTING`
- **自動編碼**：`PMRS[6碼] + {posMallCategoryName前綴}`

### 用途

設定商場特定時期對促銷活動的篩選規則。結構與 posmallitemsetting 相同，但關聯的是促銷活動而非料件。

### 核心欄位

```typescript
name                // 編號（autoKey）
startAt / endAt     // 計算起始/結束時間
posMallCategoryName // 商場類別（REF → posmallcategory）
promotionName       // 促銷活動（REF → pospromotion）
```

### 表身（LineSchema）

```typescript
storeGroups: {
  groupName  // 門市群組（REF → posstoregroup，isLineKey: true）
  effect     // 效果（ENUM: PosFilterEffect）
  memo       // 備註
}
```

### 重要腳本

**Policy（3個）**：
同 posmallitemsetting 結構（包含 `noOverlapTimeIntervalPolicy('promotionName', 'posmallpromotionsetting')`）
