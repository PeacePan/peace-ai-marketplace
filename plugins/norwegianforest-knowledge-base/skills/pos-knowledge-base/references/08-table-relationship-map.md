# 表格關聯地圖

> 本文件依據 `tracking-redundant-policy-triggers` SKILL 的五步分析法，系統性地繪製 POS 表格之間的完整關聯與觸發鏈。

## 一、外鍵依賴圖（靜態關聯）

### 核心依賴關係

```
posstore（門市）──┬──→ posstoregroup（門市群組）──→ pospromotion/poscoupon/posorderdiscount 的 storeGroups
                 │
posconfig（機台）─┼──→ posstore（門市）
                 │
posreport（日結）─┴──→ posconfig（機台） + posstore（門市）
                        ↑
                 poseguino（發票號段）──→ posconfig + posstore

possale（銷售單）
    ├── storeName   → posstore
    ├── configName  → posconfig
    ├── reportName  → posreport
    ├── memberName  → posmember
    ├── items[].promotionName → pospromotionsnapshot
    └── customerOrderName → poscustomerorder

posreturn（退貨單）
    ├── saleName    → possale
    ├── storeName   → posstore
    ├── configName  → posconfig
    └── reportName  → posreport

poscustomerorder（客訂單）
    ├── saleName    → possale
    └── storeName   → posstore

poscustomerorderwithdraw（客訂提領）
    └── customerOrderName → poscustomerorder

posdeposit（寄貨單）
    └── saleName    → possale

posdepositwithdraw（寄貨提領）
    └── depositName → posdeposit

posvoucherlog（禮券兌換）
    ├── voucherName → posvoucher
    ├── storeName   → posstore
    └── configName  → posconfig

poscouponcode（折扣序號）
    ├── generatorName → poscouponcodegenerator
    ├── couponName    → poscoupon
    └── saleName      → possale（使用後填入）

poscouponcodegenerator（批次產生器）
    └── couponName  → poscoupon

poscouponsnapshot（折扣碼快照）
    └── 繼承自 poscoupon（dependencyTables）

pospromotionsnapshot（促銷快照）
    └── 繼承自 pospromotion（dependencyTables）

posorderdiscountsnapshot（整單折扣快照）
    └── 繼承自 posorderdiscount（dependencyTables）

positemcollection（料件集合）
    └── eventName → pospromotion/poscoupon/posorderdiscount/posfreebie/posvoucher

pospet（寵物）
    ├── memberName   → posmember
    └── species      → pospetspecies

posstoreshelf（台帳）
    └── storeName    → posstore（unique）

posmallitemsetting（商場料件設定）
    ├── posMallCategoryName → posmallcategory
    ├── itemName            → ITEM
    └── storeGroups[].groupName → posstoregroup

posmallpromotionsetting（商場促銷設定）
    ├── posMallCategoryName → posmallcategory
    ├── promotionName       → pospromotion
    └── storeGroups[].groupName → posstoregroup

expenditure（支出單）
    ├── storeName   → posstore
    ├── configName  → posconfig
    └── reportName  → posreport

pospilot（前導測試）
    └── stores[].storeName → posstore

posshortcut（鍵盤群組）
    └── allowStoreGroups[].groupName → posstoregroup
```

---

## 二、業務流程觸發鏈（動態關聯）

### 2.1 結帳流程（最複雜觸發鏈）

```
【使用者操作】possale INSERT（建立銷售單）
    │
    ├─ BeforeInsert → 計算 totalTax
    ├─ Policy → 驗證金額、快照、發票、付款
    └─ [status=未完成]

【使用者操作】possale.finishCheckout（完成結帳）
    │
    ├─ Pos.pickEGUINo()
    │   └─ UPDATE poseguino.numberInterval[].offset += 1
    │
    ├─ UPDATE possale.status = 已付款
    │   └─ BeforeUpdate → 觸發 policy
    │
    ├─ 若有換購：UPDATE possale（原單/新單）
    │
    ├─ Pos.getApportedItems() → 計算料件攤值
    │
    ├─ 點加金 → MemberRights.pointChange()
    │   └─ INSERT pointaccount（點數帳本）
    │
    ├─ 禮物卡 → MemberRights.pointCardChange()
    │
    ├─ TodoJob: possale/扣庫v2
    │   └─ 分批 Inventory.pushBalance() 扣庫
    │
    ├─ TodoJob: posmember/完成結帳後更新會員資料
    │   ├─ UPDATE posmember（消費金額/次數/等級）
    │   └─ UPDATE posmember.emSyncStatus = 待同步
    │       └─ 觸發 Emarsys 同步流程（emcontacttask）
    │
    ├─ TodoJob: posmember/更新會員品類標籤
    │   └─ UPDATE posmember（品類標籤欄位）
    │
    └─ TodoJob: possale/價值攤算（若有促銷細項）
        └─ 計算折扣在料件間的攤提比例
```

### 2.2 退貨流程觸發鏈

```
【使用者操作】posreturn INSERT（建立退貨單）
    │
    ├─ Policy 驗證
    └─ 自動寫入 possale 的退貨記錄

【使用者操作】posreturn UPDATE（確認退款）
    │
    ├─ Policy 驗證退款金額
    │
    ├─ MemberRights.refundSalePoints()
    │   └─ 收回贈點（INSERT pointaccount 負值）
    │
    └─ TodoJob: posreturn/回補庫存v2
        └─ Inventory.pushBalance()（正數回補）
```

### 2.3 促銷活動修改觸發鏈

```
【使用者操作】pospromotion INSERT/UPDATE
    │
    ├─ Policy 驗證（283行）
    │   └─ Pos.insertSnapshot(ctx, 'pospromotion', record)
    │       └─ INSERT pospromotionsnapshot（建立快照）
    │
    ├─ BeforeUpdate → 更新 itemCollectionStatus = 待更新
    │
    └─ Cron（HALF_HOURLY）: 活動料件集合檔刷新
        └─ TodoJob: positemcollection/syncItemsTodoJob
            └─ UPDATE positemcollection.items（比對並同步料件）
```

### 2.4 客訂單流程觸發鏈

```
【使用者操作】poscustomerorder INSERT
    │
    ├─ Policy 驗證（150行）
    │   ├─ 確認 saleName 有效
    │   └─ 自動建立 TransferOrder（調撥單）
    │
    └─ 狀態: 待調貨

【調貨到達】poscustomerorder UPDATE（狀態 → 待提領）
    │
    └─ 可建立 poscustomerorderwithdraw

【使用者操作】poscustomerorderwithdraw INSERT
    │
    ├─ Policy 驗證
    └─ Cron（FIVE_MINS）: 扣庫排程
        └─ 扣除庫存
```

### 2.5 折扣碼批次產生觸發鏈

```
【使用者操作】poscouponcodegenerator INSERT
    │
    ├─ Policy 驗證（coupon 必須為批次發行類型）
    │
    └─ TodoJob: 產生序號
        ├─ 迴圈: INSERT poscouponcode（每次一筆）
        └─ UPDATE poscouponcodegenerator.generatedCount += 1
```

---

## 三、跨表更新的防迴圈機制

以下跨表更新必須使用 `userContent` 旗標防止無限迴圈：

| 跨表更新場景 | 防迴圈旗標 |
|------------|----------|
| posreturn → possale（寫入退貨記錄） | `PosUserContent.skipReturnUpdate` |
| posreport → posconfig（更新 reportName） | `PosUserContent.skipConfigUpdate` |
| poscustomerorder → TransferOrder | `TransferorderUserContent.skipBeforeUpdate` |
| possale → possale（換購雙向更新） | `PosUserContent.skipExchangeUpdate` |

---

## 四、依據 tracking-redundant-policy-triggers 的觸發鏈分析法

### 步驟一：確認操作邊界

以「建立 possale」為例：
- **輸入**：possale INSERT
- **直接觸發**：BeforeInsert + Policy + 資料庫寫入

### 步驟二：繪製第一層觸發

```
possale INSERT
    ├── BeforeInsert: calculateTax
    └── Policy: 驗證 → 通過 → 寫入 DB
```

### 步驟三：遞迴展開

```
possale INSERT
    ├── BeforeInsert: calculateTax → 無跨表操作
    └── Policy:
        ├── Pos.checkSnapshot() → 讀取 pospromotionsnapshot（唯讀，無觸發）
        ├── 驗證 storeName → 讀取 posstore（唯讀，無觸發）
        └── 驗證 configName → 讀取 posconfig（唯讀，無觸發）
```

### 步驟四：標記必要性

| 觸發 | 必要性 | 類型 |
|-----|-------|------|
| calculateTax | ✅ 必要 | BeforeInsert |
| Pos.checkSnapshot() | ✅ 必要 | Policy 驗證 |
| 讀取 posstore/posconfig | ✅ 必要（唯讀） | Policy 驗證 |

### 步驟五：識別冗餘觸發

以「possale 更新多個欄位」為例：
- ❌ 冗餘：每次欄位更新都觸發完整 policy 驗證（包含 checkSnapshot）
- ✅ 修正：確認哪些欄位更新不需要重新驗證快照（例如：更新 `invoiceSyncStatus` 欄位不需要重新 checkSnapshot）

---

## 五、常見跨表問題排查流程

### 問題：結帳後庫存沒有扣減

1. 查看 `possale.balanceStatus`（若欄位存在）
2. 確認 `possale/扣庫v2` TodoJob 是否建立（查 todojob 表）
3. 確認 jobQueues 有沒有卡住（查 todojob 狀態）
4. 查看 Inventory balance 記錄是否有插入

### 問題：促銷折扣計算不符

1. 確認 `possale.items[].promotionName` 是否正確關聯到 `pospromotionsnapshot`
2. 讀取 `pospromotionsnapshot` 的欄位值是否與預期相符
3. 確認 `Pos.checkSnapshot()` 有無拋出版本不符錯誤
4. 查看最新的 `pospromotion` 修改時間，確認快照是否有更新

### 問題：客訂單沒有建立調撥單

1. 確認 `poscustomerorder.transferDetails` 是否有記錄
2. 查看 policy.ts 中的調撥單建立邏輯
3. 確認 TransferOrder 表格的 policy 有無錯誤（影響 INSERT）
4. 查看相關的 userContent 旗標是否正確設定
