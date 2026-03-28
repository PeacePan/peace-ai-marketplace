# 優惠券系統

優惠券系統由三張表格組成，形成「活動定義 → 實例發放 → 批次指定發放」的架構。

## 架構概覽

```
petparkcouponevent（優惠券活動）
    ↓ 定義優惠券的類型、發放條件、使用條件
    ├── petparkcoupon（會員優惠券）
    │     - 每個發放給特定會員的優惠券實例
    │     - couponEventName → petparkcouponevent.name
    └── petparkcouponassign（優惠券指定發放）
          - 批次指定會員名單發放
          - couponName → petparkcouponevent.name
          - 執行後 → 建立多筆 petparkcoupon 記錄
```

---

## petparkcouponevent 優惠券活動檔

**TABLE_NAME**: `PET_PARK_COUPON_EVENT`
**檔案路徑**: `tables/memberRights/petparkcouponevent.ts`

### 用途

定義優惠券活動的所有規則，包含優惠券類型（折價/折扣/商品券）、發放條件（時機/身分/門市/料件）、使用條件（渠道/金額門檻/折扣設定）。

### 欄位群組

| 群組 | 主要欄位 |
|-----|---------|
| 基本資料 | name、displayName、startAt、endAt、description、magentoId、linkParams、image、department、activityType、couponEventCode |
| 發放條件設定 | couponDisplayName、type（優惠券類型）、issueType、issueChannel、timing（觸發時機）、issueTotal、validDays/validStartAt/validEndAt |
| 使用條件設定 | useChannel、applyTotal、totalLimit、discount、percent、discountBase、discountPercent、issueTotalAccumulate |

### 優惠券類型（PetParkCouponEventType）

| 類型 | 說明 |
|-----|-----|
| 折價券 | 固定金額折抵 |
| 折扣券 | 百分比折扣 |
| 商品券 | 指定商品兌換 |

### 觸發時機（timing）

決定系統何時自動發放優惠券給會員：
- 消費時（possale/posservicesale 完成後）
- 生日月
- 會員等級異動
- 指定名單（手動透過 petparkcouponassign 發放）
- 其他時機...

### 表身（lines）

| 表身 | 說明 |
|-----|-----|
| `issueMemberIdentity` | 發送適用身分（會員等級） |
| `issuePayments` | 發送適用付款別 |
| `issueStoreGroups` | 發送適用門市群組 |
| `issueItemFilters` | 發送料件篩選器 |
| `issueServiceItemFilters` | 發送服務料件篩選器 |
| `issuePackageServiceItemFilters` | 發送包月服務篩選器 |
| `useStoreGroups` | 使用適用門市群組 |
| `useItemFilters` | 使用料件篩選器 |
| `useServiceItemFilters` | 使用服務料件篩選器 |
| `usePackageServiceItemFilters` | 使用包月服務篩選器 |

### 腳本掛勾

| 掛勾 | 說明 |
|-----|-----|
| BEFORE_UPDATE | 更新前驗證（`scripts/petparkcouponevent/beforeUpdate.ts`） |
| 多個 POLICY | 門市群組、料件篩選器各自的驗證 |

---

## petparkcoupon 會員優惠券檔

**TABLE_NAME**: `PET_PARK_COUPON`
**檔案路徑**: `tables/memberRights/petparkcoupon.ts`
**腳本路徑**: `tables/memberRights/scripts/petparkcoupon/`
**JobQueues**: 20 個 queue，各 8 並發（高並發，頻繁發放）

### 用途

每筆代表一張發放給特定會員的優惠券實例，記錄發放時間、可用期間、使用狀態。

### 主要欄位

| 欄位 | 說明 |
|-----|-----|
| `displayName` | 優惠券名稱 |
| `memberName` | 歸屬會員編號 |
| `type` | 類型（ELECTRONIC 電子券 / 其他） |
| `discount` | 折價金額（可手動調整，因歷史計算問題） |
| `startAt` | 可用起日 |
| `endAt` | 可用迄日 |
| `couponEventName` | 優惠券活動編號（參照 petparkcouponevent） |
| `code` | 券號（系統自動產生，unique） |
| `sourceTable` | 來源表格（possale/posservicesale/mashoporder/petparkcouponassign） |
| `sourceRecordName` | 來源紀錄編號 |
| `usedAt` | 使用時間（核銷後填入） |
| `usedSaleName` | 使用的銷售單編號 |

### 腳本掛勾

| 掛勾 | 說明 |
|-----|-----|
| BATCH_POLICY | 批次發放驗證（`batchPolicy.ts`） |
| BEFORE_INSERT | 新增前處理，自動產生券號（`beforeInsert.ts`） |

### 核心函式

| 函式 | 說明 |
|-----|-----|
| 核銷優惠券 | 給 Magento 使用，核銷指定券 |
| 發放優惠券 | 依時機篩選活動，發放給指定會員 |
| 發放優惠券-事件版 | 同上，透過事件版 |
| 返還優惠券 | 退款時返還已使用的優惠券 |
| 類型補值 | 補填 type 欄位（舊資料補值） |
| 折價金額補算（isPublic） | 重算折價金額 |
| 封存重複發放（onJobRetry: 100） | 處理重複發放的情況 |
| 標記會員優惠券狀態（onJobRetry: 2） | 更新 posmember 的優惠券欄位 |

### 排程

| 排程 | 頻率 | 說明 |
|-----|-----|-----|
| 類型補值排程 | TEN_MINS | 補填舊券的類型欄位 |

### 事件

| 事件 | 說明 |
|-----|-----|
| `maIssueCoupon` | 處理發放優惠券事件（事件版觸發） |

---

## petparkcouponassign 優惠券指定發放檔

**TABLE_NAME**: `PET_PARK_COUPON_ASSIGN`
**檔案路徑**: `tables/memberRights/petparkcouponassign.ts`
**腳本路徑**: `tables/memberRights/scripts/petparkcouponassign/`
**JobQueues**: 5 個 queue，各 1 並發

### 用途

批次指定會員名單，在指定時間發放特定優惠券活動的券。只適用於 `timing = 指定名單` 的優惠券活動。

### 主要欄位

| 欄位 | 說明 |
|-----|-----|
| `issueAt` | 發送時間 |
| `couponName` | 優惠券活動編號（只限 timing = 指定名單） |
| `cronStatus` | 排程狀態：未執行/未完成/已完成/錯誤/已取消 |
| `cronError` | 排程錯誤訊息 |
| `processedLineId` | 最後發放表身列編號（斷點續傳） |

### 表身

| 表身 | 說明 |
|-----|-----|
| `assignList` | 發放列表（memberName → posmember） |

### 核心函式

| 函式 | 說明 |
|-----|-----|
| 優惠券發放 | 批次對表身列表發放優惠券 |
| 取消發放（isPublic） | 取消待發放任務 |

### 排程

| 排程 | 頻率 | 說明 |
|-----|-----|-----|
| 優惠券發放排程 | FIVE_MINS | 定期執行待發放任務 |

---

## 優惠券發放完整流程

### 自動發放（消費觸發）

```
possale/posservicesale 完成結帳
    ↓
posmember.issueConsumptionCoupon（FIVE_MINS 排程）
    ↓
讀取 petparkcouponevent（timing = 消費時，且符合門市/料件/金額條件）
    ↓
建立 petparkcoupon（memberName, couponEventName, startAt, endAt, code...）
    ↓
petparkcoupon.beforeInsert → 自動產生券號
    ↓
petparkcoupon.標記會員優惠券狀態 → 更新 posmember.couponStatus
```

### 批次指定發放

```
行銷人員建立 petparkcouponassign
    ↓ 指定 issueAt（發放時間）
    ↓ 選擇 couponName（必須是 timing = 指定名單 的活動）
    ↓ 匯入 assignList（會員名單）
    ↓
FIVE_MINS 排程 → 優惠券發放 function
    ↓
逐筆讀取 assignList → 建立 petparkcoupon
    ↓
支援斷點續傳（processedLineId 記錄進度）
```
