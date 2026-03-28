---
name: pos-knowledge-base
description: >
  NorwegianForest POS 系統完整業務知識庫。涵蓋 33 張 POS 表格定義（possale、posreturn、pospromotion、poscoupon、posorderdiscount、posfreebie、posvoucher、poscustomerorder、posdeposit、posstore 等）的欄位結構、狀態機、Policy/Hook/Cron 腳本邏輯、跨表關聯地圖、快照機制、以及 TodoJob 排程模式。負責 POS 相關功能（含會員結帳、促銷活動、折扣碼、客訂寄貨、日結）的開發與維護時必須載入此技能。
---

# NorwegianForest POS 知識庫

## 技能概述

本技能涵蓋 NorwegianForest 專案中所有 POS（銷售點）相關的業務知識，包含：

- **33 張 POS 表格**的完整欄位定義、狀態機、腳本掛勾
- **跨表關聯地圖**：觸發鏈、外鍵依賴、快照機制
- **Policy / Hook / Cron / TodoJob** 腳本的業務邏輯摘要
- **常見開發場景**的定位指引

---

## 參考文件目錄

| 文件 | 說明 | 何時閱讀 |
|------|------|---------|
| [01-architecture-overview.md](./references/01-architecture-overview.md) | 系統架構總覽、33 張表格速查表、目錄結構 | 首次接觸 POS 系統、需要快速定位表格 |
| [02-transaction-tables.md](./references/02-transaction-tables.md) | 核心交易表格：possale、posreturn、posreport、posconfig、posstore | 開發結帳、退貨、日結、機台、門市相關功能 |
| [03-promotion-tables.md](./references/03-promotion-tables.md) | 促銷活動表格：pospromotion、poscoupon、posorderdiscount、posfreebie、posvoucher | 開發促銷、折扣碼、加贈、禮券相關功能 |
| [04-order-logistics-tables.md](./references/04-order-logistics-tables.md) | 訂單物流表格：poscustomerorder、posdeposit、客訂提領、寄貨提領、禮券兌換紀錄 | 開發客訂單、寄貨單、提領相關功能 |
| [05-config-support-tables.md](./references/05-config-support-tables.md) | 配置支援表格：poseguino、posstoregroup、pospilot、posshortcut、expenditure、商場設定 | 開發配置管理、商場設定、支出單相關功能 |
| [06-member-related-tables.md](./references/06-member-related-tables.md) | 會員相關表格：pospet、pospetspecies、posprintablecoupon、positemcollection、posstoreshelf | 開發寵物、小白單、料件集合、台帳相關功能 |
| [07-snapshot-tables.md](./references/07-snapshot-tables.md) | 快照表格：poscouponsnapshot、pospromotionsnapshot、posorderdiscountsnapshot | 理解快照機制、排查快照相關問題 |
| [08-table-relationship-map.md](./references/08-table-relationship-map.md) | 完整表格關聯地圖、外鍵依賴圖、觸發鏈分析 | 排查跨表觸發問題、效能問題分析、影響範圍評估 |
| [09-scripts-and-policies.md](./references/09-scripts-and-policies.md) | Policy/BeforeInsert/BeforeUpdate/BatchBeforeUpdate 腳本業務邏輯摘要 | 開發或修改現有腳本時閱讀 |
| [10-todojob-cron-patterns.md](./references/10-todojob-cron-patterns.md) | POS 系統的 TodoJob 排程模式、Cron 清單、分批扣庫模式 | 開發排程功能、分析排程問題 |

---

## 快速定位指引

### 依功能找表格

| 業務需求 | 主要表格 | 腳本位置 |
|---------|---------|---------|
| 結帳銷售 | `possale` | `tables/pos/scripts/possale/` |
| 退貨處理 | `posreturn` | `tables/pos/scripts/posreturn/` |
| 日結帳務 | `posreport` | `tables/pos/scripts/posreport/` |
| 收銀機管理 | `posconfig` | `tables/pos/scripts/posconfig/` |
| 門市管理 | `posstore` | `tables/pos/scripts/posstore/` |
| 商品促銷 | `pospromotion` | `tables/pos/scripts/pospromotion/` |
| 折扣碼活動 | `poscoupon` | `tables/pos/scripts/poscoupon/` |
| 整單折扣 | `posorderdiscount` | `tables/pos/scripts/posorderdiscount/` |
| 加贈活動 | `posfreebie` | `tables/pos/scripts/posfreebie/` |
| 截角禮券 | `posvoucher` | `tables/pos/scripts/posvoucher/` |
| 客訂單管理 | `poscustomerorder` | `tables/pos/scripts/poscustomerorder/` |
| 寄貨單管理 | `posdeposit` | `tables/pos/scripts/posdeposit/` |
| 寵物管理 | `pospet` | `tables/pos/scripts/pospet/` |
| 小白單優惠券 | `posprintablecoupon` | `tables/pos/scripts/posprintablecoupon/` |
| 電子發票號碼 | `poseguino` | `tables/pos/scripts/poseguino/` |
| 支出單審核 | `expenditure` | `tables/pos/scripts/expenditure/` |

### 依問題類型找文件

| 問題類型 | 閱讀文件 |
|---------|---------|
| 某張表格的欄位定義 | `01-architecture-overview.md` → 找到表格分類 → 閱讀對應 references 文件 |
| 跨表觸發迴圈問題 | `08-table-relationship-map.md` + `function-script-context` skill |
| 排程沒有執行 | `10-todojob-cron-patterns.md` + `javacat-todojob-mechanism` skill |
| Policy 校驗規則 | `09-scripts-and-policies.md` |
| 快照版本不符 | `07-snapshot-tables.md` |
| 扣庫/回補庫存 | `02-transaction-tables.md`（possale/posreturn 的扣庫章節）|
| 促銷料件集合同步 | `03-promotion-tables.md`（positemcollection 章節）|

---

## 核心概念速覽

### POS 系統在 NorwegianForest 的定位

```
NorwegianForest（JavaCat 框架）
    └── tables/pos/           ← POS 相關的 33 張表格定義
        ├── possale.ts        ← 核心：所有銷售行為的入口
        ├── posreturn.ts      ← 退貨流程
        ├── posreport.ts      ← 日結帳務
        ├── posconfig.ts      ← 收銀機設定
        ├── posstore.ts       ← 門市主檔
        ├── pospromotion.ts   ← 促銷活動（最複雜，v80）
        ├── poscoupon.ts      ← 折扣碼
        ├── posorderdiscount.ts ← 整單折扣
        ├── posfreebie.ts     ← 加贈活動
        ├── posvoucher.ts     ← 截角禮券
        └── scripts/          ← 所有腳本（policy/hook/function/cron）
```

### 表格版本概覽（高版本 = 高複雜度）

| 版本 | 表格 |
|-----|------|
| v173 | `possale`（銷售單，最複雜） |
| v80 | `pospromotion`（促銷） |
| v46 | `posreturn`（退貨） |
| v42 | `poscustomerorder`（客訂） |
| v40 | `expenditure`（支出） |
| v39 | `positemcollection`（料件集合） |
| v32 | `posorderdiscount`（整單折扣） |
| v30 | `poscoupon`（折扣碼） |
| v26 | `posstore`（門市）、`pospromotionsnapshot` |
| v25 | `posfreebie`（加贈） |

### 三大快照表

活動修改時自動建立快照，結帳時比對快照確保一致性：
- `poscouponsnapshot` ← `poscoupon`
- `pospromotionsnapshot` ← `pospromotion`
- `posorderdiscountsnapshot` ← `posorderdiscount`

### 主要排程頻率

| 頻率 | 對應 CronRate | POS 表格使用場景 |
|-----|--------------|----------------|
| 5 分鐘 | FIVE_MINS | 寄貨單/客訂提領扣庫、禮券扣庫 |
| 10 分鐘 | TEN_MINS | 銷售單/退貨單扣庫回補 |
| 30 分鐘 | HALF_HOURLY | 促銷料件集合同步、門市NetSuite同步 |
| 每日 | DAILY | 禮券活動料件同步 |
