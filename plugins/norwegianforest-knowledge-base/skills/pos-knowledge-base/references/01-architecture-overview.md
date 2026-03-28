# POS 系統架構總覽

## 目錄結構

```
NorwegianForest/
└── tables/pos/
    ├── possale.ts                    # 銷售單（v173，最核心）
    ├── posreturn.ts                  # 退貨單（v46）
    ├── posreport.ts                  # 日結單（v10）
    ├── posconfig.ts                  # 收銀機設定（v3）
    ├── posstore.ts                   # 門市主檔（v26）
    ├── posstoregroup.ts              # 門市群組（v2）
    ├── pospilot.ts                   # 前導測試種子店（v6）
    ├── posshortcut.ts                # 鍵盤群組（v2）
    ├── poseguino.ts                  # 電子發票號碼（v5）
    ├── posconfig.ts                  # 收銀機設定（v3）
    ├── expenditure.ts                # 支出單（v40）
    │
    ├── pospromotion.ts               # 商品促銷（v80）
    ├── pospromotionsnapshot.ts       # 促銷快照（v26）
    ├── poscoupon.ts                  # 折扣碼（v30）
    ├── poscouponsnapshot.ts          # 折扣碼快照（v17）
    ├── poscouponcode.ts              # 折扣序號（v4）
    ├── poscouponcodegenerator.ts     # 折扣碼批次產生（v1）
    ├── posorderdiscount.ts           # 整單折扣（v32）
    ├── posorderdiscountsnapshot.ts   # 整單折扣快照（v18）
    ├── posfreebie.ts                 # 加贈活動（v25）
    ├── posvoucher.ts                 # 截角禮券活動（v16）
    ├── posvoucherlog.ts              # 禮券兌換紀錄（v9）
    ├── positemcollection.ts          # 料件活動集合（v39）
    │
    ├── poscustomerorder.ts           # 客訂單（v42）
    ├── poscustomerorderwithdraw.ts   # 客訂提領單（v19）
    ├── posdeposit.ts                 # 寄貨單（v13）
    ├── posdepositwithdraw.ts         # 寄貨提領單（v8）
    │
    ├── pospet.ts                     # 寵物檔（v4）
    ├── pospetspecies.ts              # 寵物品種（無版本號）
    ├── posprintablecoupon.ts         # 小白單優惠券（v14）
    ├── posstoreshelf.ts              # 門市台帳（v5）
    ├── posmallcategory.ts            # 商場類別（v3）
    ├── posmallitemsetting.ts         # 商場料件設定（v2）
    └── posmallpromotionsetting.ts    # 商場促銷設定（v2）

    └── scripts/
        ├── possale/                  # 銷售單腳本（最多）
        ├── posreturn/                # 退貨單腳本
        ├── posconfig/                # 機台腳本
        ├── posstore/                 # 門市腳本
        ├── posreport/                # 日結單腳本
        ├── pospromotion/             # 促銷腳本
        ├── poscoupon/                # 折扣碼腳本
        ├── posorderdiscount/         # 整單折扣腳本
        ├── posfreebie/               # 加贈腳本
        ├── posvoucher/               # 禮券腳本
        ├── poscustomerorder/         # 客訂單腳本
        ├── posdeposit/               # 寄貨單腳本
        ├── poscustomerorderwithdraw/ # 客訂提領腳本
        ├── posdepositwithdraw/       # 寄貨提領腳本
        ├── positemcollection/        # 料件集合腳本
        ├── pospet/                   # 寵物腳本
        ├── poseguino/                # 電子發票號碼腳本
        ├── expenditure/              # 支出單腳本
        └── ...
```

---

## 33 張 POS 表格速查表

### 交易核心表格（5 張）

| 表格名稱 | 表格代碼 | 版本 | 用途 | 自動編碼格式 |
|---------|---------|-----|------|------------|
| 銷售單 | `POS_SALE` | v173 | 所有結帳交易的核心記錄 | {storeName前綴}YYMMDD[4碼] |
| 退貨單 | `POS_RETURN` | v46 | 退貨流程管理 | {自訂前綴}{storeName}YYMMDD[4碼] |
| 日結單 | `POS_REPORT` | v10 | 收銀機每日結帳 | {storeName前綴}YYYYMMDD[4碼] |
| 收銀機設定 | `POS_CONFIG` | v3 | 收銀機（機台）配置 | {3位門市號}{2位機台號} |
| 門市主檔 | `POS_STORE` | v26 | 門市基本資料與營運設定 | 3位數字（無自動） |

### 促銷活動表格（10 張）

| 表格名稱 | 表格代碼 | 版本 | 用途 | 自動編碼格式 |
|---------|---------|-----|------|------------|
| 商品促銷 | `POS_PROMOTION` | v80 | 料件級促銷活動 | AA[6碼]+{channelCode後綴} |
| 促銷快照 | `POS_PROMOTION_SNAPSHOT` | v26 | 促銷修改版本記錄 | 繼承自 pospromotion |
| 折扣碼 | `POS_COUPON` | v30 | 折扣碼活動管理 | CC[6碼]+{channelCode後綴} |
| 折扣碼快照 | `POS_COUPON_SNAPSHOT` | v17 | 折扣碼修改版本記錄 | 繼承自 poscoupon |
| 折扣序號 | `POS_COUPON_CODE` | v4 | 個別折扣序號管理 | autoKeyNonSerialId |
| 折扣碼批次產生 | `POS_COUPON_CODE_GENERATOR` | v1 | 批次產生折扣序號 | {couponName前綴}+[流水號] |
| 整單折扣 | `POS_ORDER_DISCOUNT` | v32 | 整筆訂單折扣活動 | BB[6碼]+{channelCode後綴} |
| 整單折扣快照 | `POS_ORDER_DISCOUNT_SNAPSHOT` | v18 | 整單折扣修改版本記錄 | 繼承自 posorderdiscount |
| 加贈活動 | `POS_FREEBIE` | v25 | 加購/贈品活動 | EE[6碼]+{channelCode後綴} |
| 截角禮券活動 | `POS_VOUCHER` | v16 | 截角禮券活動配置 | DD[6碼]+{channelCode後綴} |

### 訂單物流表格（5 張）

| 表格名稱 | 表格代碼 | 版本 | 用途 | 自動編碼格式 |
|---------|---------|-----|------|------------|
| 客訂單 | `POS_CUSTOMER_ORDER` | v42 | 客戶預訂管理 | CO{storeName前綴}YYMMDD[3碼] |
| 客訂提領單 | `POS_CUSTOMER_ORDER_WITHDRAW` | v19 | 客訂商品提領 | CW{storeName前綴}YYMMDD[3碼] |
| 寄貨單 | `POS_DEPOSIT` | v13 | 商品預存管理 | DE[6碼] |
| 寄貨提領單 | `POS_DEPOSIT_WITHDRAW` | v8 | 寄貨商品提領 | DW[6碼] |
| 禮券兌換紀錄 | `POS_VOUCHER_LOG` | v9 | 截角禮券兌換記錄 | [流水號8碼] |

### 配置支援表格（8 張）

| 表格名稱 | 表格代碼 | 版本 | 用途 | 自動編碼格式 |
|---------|---------|-----|------|------------|
| 門市群組 | `POS_STORE_GROUP` | v2 | 門市分組管理 | 手動輸入名稱 |
| 前導測試種子店 | `POS_PILOT` | v6 | 功能前導測試控制 | PP[5碼] |
| 鍵盤群組 | `POS_SHORTCUT` | v2 | POS 快捷鍵設定 | [流水號8碼] |
| 電子發票號碼 | `POS_EGUI_NO` | v5 | 電子發票號碼區段管理 | [流水號] |
| 支出單 | `EXPENDITURE` | v40 | 門市支出申請與審核 | [流水號] |
| 商場類別 | `POS_MALL_CATEGORY` | v3 | 商場料件類別定義 | PMC[6碼] |
| 商場料件設定 | `POS_MALL_ITEM_SETTING` | v2 | 商場料件篩選設定 | PMIS[6碼]+{類別前綴} |
| 商場促銷設定 | `POS_MALL_PROMOTION_SETTING` | v2 | 商場促銷篩選設定 | PMRS[6碼]+{類別前綴} |

### 會員與其他表格（5 張）

| 表格名稱 | 表格代碼 | 版本 | 用途 | 自動編碼格式 |
|---------|---------|-----|------|------------|
| 寵物檔 | `POS_PET` | v4 | 會員寵物資料 | [流水號8碼] |
| 寵物品種 | `POS_PET_SPECIES` | — | 寵物品種定義 | 手動輸入名稱 |
| 小白單優惠券 | `POS_PRINTABLE_COUPON` | v14 | 結帳後紙本優惠券設定 | MK[6碼] |
| 料件活動集合 | `POS_ITEM_COLLECTION` | v39 | 活動包含料件的同步快取 | {eventName前綴}YYMMDD[2碼] |
| 門市台帳 | `POS_STORE_SHELF` | v5 | 門市商品陳列台帳 | 手動命名 |

---

## 系統架構圖

```
POS 系統架構
═══════════════════════════════════════════════════════════════

【入口】
  POS 收銀機（Ragdoll 前端）
      ↓
  posconfig（機台）→ posstore（門市）→ posreport（日結）

【交易核心】
  possale（銷售單）
      ├── 讀取促銷快照（checkSnapshot）
      ├── 計算折扣（pospromotion/poscoupon/posorderdiscount）
      ├── 完成結帳（finishCheckout）
      │   ├── 電子發票取號（poseguino）
      │   ├── 觸發 TodoJob 更新會員資料（posmember）
      │   ├── 觸發 TodoJob 扣庫（Inventory）
      │   └── 觸發 TodoJob 小白單（posprintablecoupon）
      └── 關聯退貨（posreturn）

【促銷系統】
  pospromotion ──→ pospromotionsnapshot
  poscoupon    ──→ poscouponsnapshot
                    ↑
  posorderdiscount ──→ posorderdiscountsnapshot
      ↑
  positemcollection（同步各活動的料件清單）

【訂單物流】
  possale ──→ posdeposit（寄貨）──→ posdepositwithdraw（提領）
          └──→ poscustomerorder（客訂）──→ poscustomerorderwithdraw（提領）

【折扣碼】
  poscoupon ──→ poscouponcodegenerator ──→ poscouponcode（序號）

【禮券】
  posvoucher ──→ posvoucherlog（兌換紀錄）

【會員相關（POS 側）】
  pospet（寵物）→ pospetspecies（品種）
  posprintablecoupon（小白單）

【配置管理】
  posstore ──→ posstoregroup（門市群組）
          └──→ posstoreshelf（台帳）
  pospilot（功能前導測試）
  poseguino（電子發票號段）
  expenditure（支出單）
```

---

## 表格分類與腳本類型對照

### 有 jobQueues（非同步排程）的表格

| 表格 | jobQueues 設定 | 主要排程用途 |
|------|--------------|------------|
| `possale` | '1,1,1,1,1,1,1,1,1,1' (10個) | 扣庫、價值攤算、會員更新 |
| `posreturn` | '1,1,1' (3個) | 庫存回補、發票取消、會員點數扣回 |
| `posreport` | '1,1,1,1,1,1,1,1,1,1' (10個) | 結帳、NetSuite 同步 |
| `posstore` | '1,1,1' (3個) | NetSuite 同步 |
| `pospromotion` | '1,1,1,1,1' (5個) | 料件集合同步、門市上限刷新、Foodomo 同步 |
| `poscoupon` | '1,1,1' (3個) | 料件集合同步 |
| `posorderdiscount` | '1,1,1,1,1' (5個) | 料件集合同步、門市上限刷新 |
| `posfreebie` | '1,1,1,1,1' (5個) | 料件集合同步、門市上限刷新 |
| `posvoucher` | '1' (1個) | 料件集合同步 |
| `poscustomerorder` | '1,1,1,1,1' (5個) | 扣庫、調撥單建立 |
| `poscustomerorderwithdraw` | '1' (1個) | 扣庫 |
| `posdepositwithdraw` | CronRate.FIVE_MINS | 扣庫 |
| `poscouponcodegenerator` | '1,1,1,1,1' (5個) | 批次產生序號 |
| `posdeposit` | '1' (1個) | 庫存回補 |
| `positemcollection` | '1,1,1,1,1' (5個) | 料件同步 |
| `posvoucherlog` | '1,1,1,1' (4個) | 扣庫、NetSuite 同步 |

### 腳本類型清單

```
Policy（業務規則校驗）：
    possale, posreturn, posconfig, posstore, posreport, pospromotion, poscoupon,
    posorderdiscount, posfreebie, posvoucher, poscustomerorder, posdeposit,
    poseguino, poscouponcodegenerator, expenditure, posshortcut, posmallitemsetting,
    posmallpromotionsetting, pospilot, pospet

BeforeInsert（新增前自動處理）：
    possale（計算稅額）、poscoupon（設定itemCollectionStatus）、
    poseguino（驗證號碼格式）、posdeposit（BATCH_BEFORE_INSERT）、
    posreport（關聯posconfig）、pospet（設定預設值）

BeforeUpdate（更新前自動處理）：
    possale、poscoupon、posorderdiscount、posfreebie、poseguino

Functions（業務函式）：
    possale: finishCheckout, balance（扣庫）, getApportionItems, apportion（攤值）
    posreturn: refund, cancel, finish, balance（回補庫存）
    poscouponcode: redeemCode, redeemCodeV2（核銷）
    posstoreshelf: getNewestNoList, 台帳生效通知, 下載台帳圖
    positemcollection: syncItems, syncItemsTodoJob（料件同步）
    posreport: closeEntry（日結）
    posdeposit: balance（回補庫存）

Cron（定時排程）：
    詳見 10-todojob-cron-patterns.md
```
