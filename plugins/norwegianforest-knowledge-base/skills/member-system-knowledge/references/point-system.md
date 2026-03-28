# 點數系統

## pointaccount 點數帳本檔

**TABLE_NAME**: `POINT_ACCOUNT`
**檔案路徑**: `tables/memberRights/pointaccount.ts`
**腳本路徑**: `tables/memberRights/scripts/pointaccount/`
**JobQueues**: 10 個 queue，各 1 並發
**說明**: 每次點數異動都新增一筆，以帳本方式累計

### 主要欄位

| 欄位 | 說明 |
|-----|-----|
| `memberName` | 會員編號（參照 posmember） |
| `type` | 類型（PointHistoryType 列舉） |
| `sourceTable` | 來源表格名稱（possale/posreturn/importpoint/... 等） |
| `sourceRecordName` | 來源紀錄編號 |
| `maOrderId` | Magento 訂單編號 |
| `changeAmount` | 異動點數（正為增加，負為扣除） |
| `localBalance` | 本筆紀錄剩餘點數（僅新增類型時有值） |
| `balance` | 會員目前剩餘點數（截至本筆的總餘額） |
| `expiredAt` | 到期時間 |
| `memo` | 備註（對消費者顯示） |
| `sysMemo` | 系統備註（內部用） |

### 點數來源（sourceTable 可能值）

```
possale           - POS 銷售單（消費贈點）
posreturn         - POS 銷退單（退貨扣點）
posservicesale    - 服務銷售單
posservicereturn  - 服務銷退單
importpoint       - 批次匯入點數
mashoporder       - Magento 訂單
mashoprefund      - Magento 退款
posmember         - 直接操作會員（如重算）
mapartialorder    - 分批取訂單
mapartialrefund   - 分批取退款
feversocialpoints - FeverSocial 點數歸戶
```

### 表身（history）

每筆點數帳本紀錄下的歷程，記錄該筆的每次異動細節：
- `type`、`changeAmount`、`currentBalance`、`memo`、`sourceTable`、`sourceRecordName`

### 核心函式

| 函式 | 說明 |
|-----|-----|
| 點數異動 | 已無使用（保留供測試參考） |
| 點數回補（isPublic） | 對會員某筆單據進行點數回補 |
| 過期點數處理（onJobRetry: 3） | 扣除已過期的點數 |
| 核銷點數 | 給 Magento/LocalPOS 使用 |
| 返還點數 | 給 Magento 使用（退還整筆訂單點數） |
| 封存指定的點數帳本檔 | 封存已派發點數 |

### 排程

| 排程 | 頻率 | 說明 |
|-----|-----|-----|
| 過期點數處理排程 | FIVE_MINS (prod) | 定期扣除到期點數 |
| 已派發點數批次封存 | FIVE_MINS (prod only) | 封存已使用的點數記錄 |

---

## givepointsetting 贈點活動設定檔

**TABLE_NAME**: `GIVE_POINT_SETTING`
**檔案路徑**: `tables/memberRights/givepointsetting.ts`
**腳本路徑**: `tables/memberRights/scripts/givepointsetting/`

### 用途

定義贈點規則，支援多種類型（固定贈點、消費比例、倍數等），並可依門市群組、時間（每週/每月特定日）、互斥設定進行篩選。

### 贈點類型（GivePointType）

活動類型決定贈點計算方式：
- 固定贈點
- 消費比例（消費 100 元得 N 點）
- 點數倍數（N 倍點數）
- 禮物卡

### 主要欄位

| 欄位 | 說明 |
|-----|-----|
| `name` | 贈點活動編號（前綴 GPS） |
| `type` | 活動類型 |
| `displayName` | 活動名稱 |
| `startAt` / `endAt` | 活動時間 |
| `expireDays` | 點數到期天數 |
| `expiredAt` | 點數到期日（與 expireDays 二擇一） |
| `orderAmountMin` | 消費訂單滿額門檻 |
| `memberUsedMax` | 每個會員最多使用組數 |
| `allStoreMax` | 全門市最多使用組數 |
| `allStoreUsed` | 已使用的全門市組數 |

### 表身（lines）

| 表身 | 說明 |
|-----|-----|
| `classicSetting` | 一般會員發送設定（channel/base/points/倍數/上限） |
| `goldSetting` | 金卡會員發送設定 |
| `blackSetting` | 黑卡會員發送設定 |
| `itemFilters` | 料件篩選器（適用哪些商品） |
| `storeGroups` | 門市群組設定（允用/禁用門市） |
| `weekly` | 周適用日（1=周一, 7=周日） |
| `monthly` | 月適用日（1~31） |
| `exclusiveEvent` | 互斥活動（此活動被套用後，互斥活動不可再套） |

### 核心函式

| 函式 | 說明 |
|-----|-----|
| 點數禮物卡異動 | 對會員進行禮物卡點數異動 |
| 使用組數異動（onJobRetry: 3） | 更新已使用組數 |

---

## importpoint 匯入點數檔

**TABLE_NAME**: `IMPORT_POINT`
**檔案路徑**: `tables/memberRights/importpoint.ts`
**腳本路徑**: `tables/memberRights/scripts/importpoint/`
**JobQueues**: 5 個 queue，各 1 並發

### 用途

批次指定會員名單，在預約時間發送固定點數。支援設定到期天數或到期日。

### 主要欄位

| 欄位 | 說明 |
|-----|-----|
| `points` | 贈送點數 |
| `expireDays` | 點數到期天數（與 expiredAt 二擇一） |
| `expiredAt` | 點數到期日期 |
| `reservedAt` | 預約匯入時間 |
| `memo` | 備註（顯示在消費者點數歷程） |
| `taskStatus` | 匯入狀態：未執行/未完成/完成/錯誤/已取消 |
| `taskError` | 匯入錯誤訊息 |
| `processedLineId` | 最後處理表身列編號（斷點續傳） |

### 表身

| 表身 | 說明 |
|-----|-----|
| `memberList` | 指定會員名單（memberName → posmember） |

### 核心函式

| 函式 | 說明 |
|-----|-----|
| 匯入點數 | 執行待處理的點數匯入任務 |
| 取消匯入（isPublic） | 取消匯入任務 |

### 排程

| 排程 | 頻率 | 說明 |
|-----|-----|-----|
| 匯入點數排程 | HOURLY | 定期執行待匯入任務 |

---

## pointdiscount 點數折現活動設定檔

**TABLE_NAME**: `POINT_DISCOUNT`
**檔案路徑**: `tables/memberRights/pointdiscount.ts`

### 用途

設定使用點數折抵訂單金額的比例，例如 10 點折抵 1 元。

### 主要欄位

| 欄位 | 說明 |
|-----|-----|
| `deductingPoint` | 扣除點數（例如 10） |
| `discount` | 折抵金額（例如 1） |
| `orderDiscountLimit` | 折抵訂單金額上限（% 數，0~100） |
| `startAt` | 開始日期 |
| `endAt` | 結束時間 |
| `department` | 歸屬部門 |
| `activityType` | 活動類型 |

---

## feversocialpoints FS 點數歸戶檔

**TABLE_NAME**: `FEVER_SOCIAL_POINTS`
**檔案路徑**: `tables/memberRights/feversocialpoints.ts`
**JobQueues**: 5 個 queue，各 1 並發

### 用途

接收 FeverSocial 點數歸戶資訊，自動發送點數給會員。由 Magento/外部系統呼叫。

### 主要欄位

| 欄位 | 說明 |
|-----|-----|
| `memberName` | 會員編號 |
| `points` | 贈送點數 |
| `expiredAt` | 點數到期日期 |
| `memo` | 備註（企業自定義識別碼） |
| `transactionId` | 交易編號 |
| `transactionType` | 交易類型 |
| `transactionPromotionName` | 活動名稱 |
| `transactionPromotionId` | 活動編號 |

### 核心函式

| 函式 | 說明 |
|-----|-----|
| 點數歸戶 | 給 Magento 使用（直接呼叫） |
| 點數歸戶-事件版 | 給 Magento 使用（透過事件） |

### 事件

| 事件 | 說明 |
|-----|-----|
| `depositPoints` | 處理點數歸戶事件（事件版觸發） |

---

## pointpromotion 點數消費活動設定檔（點加金）

**TABLE_NAME**: `POINT_PROMOTION`
**檔案路徑**: `tables/memberRights/pointpromotion.ts`
**JobQueues**: 3 個 queue，各 1 並發

### 用途

設定點加金活動，讓會員可以用點數+金額兌換指定商品。

### 主要欄位

| 欄位 | 說明 |
|-----|-----|
| `displayName` | 活動名稱 |
| `startAt` / `endAt` | 活動時間 |
| `desc` | 說明文字 |
| `img` | 活動圖片 |
| `lastCalcSaleID` | 最後更新的銷售系統編號（用於增量計算） |
| `department` | 歸屬部門 |
| `activityType` | 活動類型 |

### 表身

| 表身 | 說明 |
|-----|-----|
| `redeemItems` | 兌換品項（料件名稱、可兌換數量、所需點數/金額、門市限購） |
| `storeGroups` | 指定適用門市 |

### 核心函式

| 函式 | 說明 |
|-----|-----|
| 更新已兌換數量（onJobRetry: 3, isPublic） | 計算活動時間內銷售產生的已兌換數 |

### 排程

| 排程 | 頻率 | 說明 |
|-----|-----|-----|
| 更新已兌換數量 | HALF_HOURLY | 定期更新兌換統計 |
