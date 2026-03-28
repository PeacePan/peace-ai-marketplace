# Emarsys CRM 同步系統

Emarsys 是外部 CRM 平台，NorwegianForest 透過六張表格與其雙向溝通：
**推送資料給 Emarsys**（會員/商品/銷售/門市）以及**接收 Emarsys 回調**（Webhook 觸發發券/贈點）。

## 架構概覽

```
【NF → EM 推送】
posmember（emSyncStatus = 待同步）
    ↓ FIVE_MINS 排程（產生會員任務排程）
emcontacttask（會員任務） → CSV → Emarsys WebDAV

possale / posservicesale / posreturn
    ↓ HALF_HOURLY 排程
emsaletask（銷售任務） → CSV → Emarsys SFTP

item / serviceitem（每日 00:00）
    ↓ FIVE_MINS + DAILY 排程
emproducttask（商品任務） → CSV → Emarsys SFTP

posstore（每小時）
    ↓ HOURLY 排程
emstoretask（門市任務） → CSV → Emarsys SFTP

【EM → NF 回調】
Emarsys Webhook 請求
    ↓ BEFORE_INSERT 驗證（防重複）
emwebhook（Webhook 紀錄）
    ↓ FIVE_MINS 排程（建立任務）
    ├── behaviour = 發送優惠券 → petparkcouponassign
    └── behaviour = 贈送點數 → importpoint

Emarsys WebDAV 回傳檔案
    ↓ FIVE_MINS 排程（檔案掃描）
emfiletask（檔案任務）
    ↓ FIVE_MINS 排程（處理任務）
    ├── type = 點數 → 發放點數（pointaccount）
    └── type = 優惠券 → 發放優惠券（petparkcouponassign）
```

---

## emcontacttask 會員任務檔

**TABLE_NAME**: `EM_CONTACT_TASK`
**檔案路徑**: `tables/emarsys/emcontacttask.ts`
**腳本路徑**: `tables/emarsys/scripts/emcontacttask/`
**JobQueues**: `10,1,1,1,1,1,1,1,1,1`（第一個 queue 建立任務/匯入，其餘更新狀態）

### 用途

將 `posmember.emSyncStatus = 待同步` 的會員資料批次打包成 CSV，上傳至 Emarsys WebDAV，完成後更新 `emSyncStatus = 已完成`。

### 主要欄位

| 欄位 | 說明 |
|-----|-----|
| `members` | 此任務要上傳的會員系統編號，以逗號分隔（不放表身以節省建立時間） |
| `uploadedFile` | 已上傳到 WebDAV 的檔名 |
| `cronStatus` | 排程狀態：未執行 / 已上傳 / 待重新上傳 / 已完成 |
| `cronError` | 排程錯誤訊息 |

**cronStatus 流程說明：**
- `未執行` → 任務剛建立，尚未處理
- `已上傳` → CSV 已上傳到 WebDAV，但尚未更新 posmember.emSyncStatus
- `待重新上傳` → 上傳時發生錯誤，需重新上傳
- `已完成` → CSV 已上傳，且 emSyncStatus 已更新為「已完成」

### 核心函式

| 函式 | 說明 |
|-----|-----|
| 匯入會員資料 | 根據 members 欄位製作 CSV 並上傳到 Emarsys WebDAV |
| 更新會員同步回Em狀態 | 將任務內會員的 posmember.emSyncStatus 改為「已完成」 |
| 重新執行任務（isPublic） | 將 cronStatus 改為「未執行」，重新上傳 |

### 排程

| 排程 | 頻率 | 說明 |
|-----|-----|-----|
| 產生會員任務排程 | FIVE_MINS | 抓出 emSyncStatus = 待同步的會員，分批建立任務 |
| 重新匯入會員資料排程 | FIVE_MINS | 抓出 cronStatus = 待重新上傳的任務，重新執行 |

### 完整同步流程

```
possale 結帳完成 / posmember 相關欄位異動
    ↓
posmember.emSyncStatus 設為「待同步」
    ↓
FIVE_MINS 排程（產生會員任務排程）
    ↓ 抓出 emSyncStatus = 待同步 的所有會員
    ↓ 切分成多個批次 → 建立多筆 emcontacttask
    ↓ 建立「匯入會員資料」TodoJob
    ↓
「匯入會員資料」function
    ↓ 讀取 posmember 欄位 → 組合 CSV
    ↓ 上傳至 Emarsys WebDAV
    ↓ cronStatus = 已上傳
    ↓
「更新會員同步回Em狀態」function
    ↓ 將 members 內所有會員的 posmember.emSyncStatus = 已完成
    ↓ cronStatus = 已完成
```

---

## emfiletask Emarsys 發放排程檔（回調處理）

**TABLE_NAME**: `EM_FILE_TASK`
**檔案路徑**: `tables/emarsys/emfiletask.ts`
**腳本路徑**: `tables/emarsys/scripts/emfiletask/`
**JobQueues**: 5 個 queue，各 1 並發
**特殊設定**: `idType: IdType.NATIVE`（不自動產生 ID）

### 用途

掃描 Emarsys WebDAV 的回傳檔案，根據檔案類型執行對應操作（發放點數或優惠券）。是 Emarsys 回傳指令到 NorwegianForest 的通道之一。

### 主要欄位

| 欄位 | 說明 |
|-----|-----|
| `type` | 任務類型（EmFileTaskType：點數 / 優惠券） |
| `fileName` | 掃描到的檔案名稱 |
| `status` | 狀態（EmFileTaskStatus：PROCESSING / ...） |
| `totalCount` | 資料筆數 |
| `processedCount` | 已處理筆數 |
| `completedAt` | 完成時間 |
| `errorMessage` | 錯誤訊息 |

### 核心函式

| 函式 | 說明 |
|-----|-----|
| 發放點數 | 解析檔案，新增至 importpoint |
| 發放優惠券 | 解析檔案，新增至 petparkcouponassign |
| 重新執行 | 將 WebDAV 上的檔案標記為處理中，重新處理 |

### 排程

| 排程 | 頻率 | 說明 |
|-----|-----|-----|
| 檔案掃描排程 | FIVE_MINS | 掃描 WebDAV，發現新檔案建立 emfiletask 記錄 |
| 處理任務排程 | FIVE_MINS | 抓取 status = PROCESSING 的任務，建立 TodoJob 執行 |

---

## emwebhook Emarsys Webhook 紀錄表

**TABLE_NAME**: `EM_WEBHOOK`
**檔案路徑**: `tables/emarsys/emwebhook.ts`
**腳本路徑**: `tables/emarsys/scripts/emwebhook/`
**JobQueues**: 5 個 queue，各 1 並發

### 用途

接收來自 Emarsys Automate Program 的 Webhook 請求，根據行為（behaviour）觸發對應的優惠券發放或點數贈送。

每次 Emarsys 的自動化流程執行後，會呼叫 NorwegianForest 的 Webhook 入口，帶上 `emProgramIdV2`（程序編號）與 `contactListName`（目標會員名單名稱）。系統會查詢 Emarsys API 取得名單內所有會員的 posmember 編號，再批次執行對應操作。

### 主要欄位

| 欄位 | 說明 |
|-----|-----|
| `behaviour` | 行為（EmWebhookBehaviour：發送優惠券 / 贈送點數） |
| `emProgramIdV2` | Automate Program 編號（統一用字串儲存） |
| `contactListName` | Contact List 名稱（Emarsys 維護的目標會員名單） |
| `eventTime` | 請求時間（用於防重複：1 小時內同 Program + ContactList 不能重複） |
| `param` | Webhook 參數（JSON 字串） |
| `contactListId` | Contact List 編號（API 查詢後填入） |
| `contactListCount` | Contact List 會員筆數 |
| `cronStatus` | 排程狀態：未執行 / 不執行 / 執行中 / 已完成 / 錯誤 |
| `memo` | 備註 |

### 腳本掛勾

| 掛勾 | 說明 |
|-----|-----|
| POLICY | 驗證 Webhook 請求的合法性 |
| BEFORE_INSERT | 防止 1 小時內同 Program + ContactList 重複建立 |

### 核心函式

| 函式 | 說明 |
|-----|-----|
| 發送優惠券（onJobRetry: 3） | 呼叫 EM API 取得會員清單 → 新增至 petparkcouponassign |
| 贈送點數（onJobRetry: 3） | 呼叫 EM API 取得會員清單 → 新增至 importpoint |
| 建立任務 | 建立 TodoJob 分頁處理大量會員 |
| 任務完成檢查 | 確認所有子工作完成，更新 cronStatus = 已完成 |

**函式參數**（`arguments`）：
- `offset`、`limit`、`count`：用於分頁處理大量會員（Contact List 可能有數萬筆）

### 排程

| 排程 | 頻率 | 說明 |
|-----|-----|-----|
| 建立任務排程 | FIVE_MINS | 抓出 cronStatus = 未執行的 Webhook，建立分頁 TodoJob |
| 任務完成檢查排程 | FIVE_MINS | 確認各 TodoJob 完成，更新任務狀態 |

### Webhook 處理流程

```
Emarsys Automate Program 觸發
    ↓ HTTP POST → NorwegianForest Webhook 入口
    ↓ BEFORE_INSERT 驗證（1 小時內同 Program + ContactList 不重複）
emwebhook 新增記錄（behaviour, emProgramIdV2, contactListName）
    ↓
FIVE_MINS 排程（建立任務排程）
    ↓ 查詢 Emarsys API，取得 contactListId 與 contactListCount
    ↓ 按 limit 分頁 → 建立多個 TodoJob（發送優惠券 / 贈送點數 function）
    ↓
各分頁 function（offset/limit/count）
    ↓ 呼叫 EM API 取得該頁的會員 posmember 編號清單
    ├── behaviour = 發送優惠券 → 新增 petparkcouponassign（指定發放）
    └── behaviour = 贈送點數 → 新增 importpoint（批次匯入點數）
    ↓
FIVE_MINS 排程（任務完成檢查）
    ↓ 確認所有 TodoJob 完成
    ↓ cronStatus = 已完成
```

---

## emsaletask 銷售資料同步檔

**TABLE_NAME**: `EM_SALE_TASK`
**檔案路徑**: `tables/emarsys/emsaletask.ts`
**腳本路徑**: `tables/emarsys/scripts/emsaletask/`
**JobQueues**: 1 個 queue

### 用途

將銷售資料（possale / posreturn / posservicesale）同步到 Emarsys SFTP，供 Emarsys 進行銷售分析與個人化行銷。

### 主要欄位

| 欄位 | 說明 |
|-----|-----|
| `uploadedFile` | 上傳到 SFTP 的檔名 |
| `cronStatus` | EmSaleTaskCronStatus 列舉 |
| `cronLog` | 排程執行紀錄（多行文字） |

### 表身

| 表身 | 說明 |
|-----|-----|
| `posSaleItems` | 銷售單編號清單 |
| `posReturnItems` | 銷退單編號清單 |
| `posServiceSaleItems` | 服務銷售單編號清單 |

### 核心函式

| 函式 | 說明 |
|-----|-----|
| 自動匯出銷售資料 | 找出待上傳的銷售資料 → CSV → Emarsys SFTP |
| 手動匯出銷售資料（isPublic） | 手動觸發匯出 |
| 手動匯出銷售資料-回補（isPublic） | 補匯出遺漏的銷售資料 |

### 排程

| 排程 | 頻率 | 說明 |
|-----|-----|-----|
| 自動匯出銷售資料排程 | HALF_HOURLY | 每 30 分鐘同步最新銷售資料 |

---

## emproducttask 商品資料同步檔

**TABLE_NAME**: `EM_PRODUCT_TASK`
**檔案路徑**: `tables/emarsys/emproducttask.ts`
**腳本路徑**: `tables/emarsys/scripts/emproducttask/`
**JobQueues**: 1 個 queue

### 用途

將料件（item）和服務料件（serviceitem）資料同步到 Emarsys SFTP，供 Emarsys 個人化商品推薦使用。每次執行處理有限時間，支援斷點續傳（`lastId`）。

### 主要欄位

| 欄位 | 說明 |
|-----|-----|
| `taskTable` | 目前處理的表格（料件 → 服務料件，依序處理） |
| `lastId` | 斷點續傳的系統編號（_id） |
| `uploadedFile` | 已上傳到 SFTP 的檔案路徑 |
| `cronStatus` | 排程狀態：未執行 / 處理中 / 待上傳 / 已完成 |
| `sftpLog` | SFTP 連線紀錄 |

### 核心函式

| 函式 | 說明 |
|-----|-----|
| 執行匯入商品資料（isPublic） | 依任務建立日期的當天商品資料，產生 CSV 上傳 SFTP |

### 排程

| 排程 | 頻率 | 說明 |
|-----|-----|-----|
| 自動匯入商品資料排程 | FIVE_MINS | 持續執行直到該批商品全部處理完 |
| 自動新增商品任務排程 | DAILY 00:00 台灣 | 每天午夜建立新的商品任務 |

---

## emstoretask 門市資料同步檔

**TABLE_NAME**: `EM_STORE_TASK`
**檔案路徑**: `tables/emarsys/emstoretask.ts`
**腳本路徑**: `tables/emarsys/scripts/emstoretask/`
**JobQueues**: 1 個 queue

### 用途

將 posstore 門市資料同步到 Emarsys SFTP，供 Emarsys 門市相關個人化推薦使用。

### 主要欄位

| 欄位 | 說明 |
|-----|-----|
| `uploadedFile` | 上傳到 SFTP 的檔名 |
| `cronStatus` | 排程狀態：未執行 / 已完成 |
| `sftpLog` | SFTP 連線紀錄 |

### 核心函式

| 函式 | 說明 |
|-----|-----|
| 執行自動匯入門市資料（isPublic） | 建立日期當天的門市資料 → CSV → SFTP |
| 重置任務排程狀態（isDev） | 重置 cronStatus 為未執行，可重新執行 |

### 排程

| 排程 | 頻率 | 說明 |
|-----|-----|-----|
| 自動匯入門市資料排程 | HOURLY | 每小時同步門市資料 |

---

## Emarsys 與其他表格的關聯

| Emarsys 表格 | 依賴/影響的表格 | 說明 |
|---|---|---|
| `emcontacttask` | posmember（讀寫） | 讀取待同步會員、更新 emSyncStatus |
| `emfiletask` | importpoint（寫） | 解析回傳檔案，觸發批次點數 |
| `emfiletask` | petparkcouponassign（寫） | 解析回傳檔案，觸發批次發券 |
| `emwebhook` | petparkcouponassign（寫） | Webhook 觸發發券 |
| `emwebhook` | importpoint（寫） | Webhook 觸發贈點 |
| `emsaletask` | possale、posreturn、posservicesale（讀） | 匯出銷售資料 |
| `emproducttask` | item、serviceitem（讀） | 匯出商品資料 |
| `emstoretask` | posstore（讀） | 匯出門市資料 |

---

## 注意事項

1. **emcontacttask 的 members 欄位**：
   會員名單以逗號分隔存在單一字串欄位（非表身），這是為了加快任務建立速度，避免大量 push 表身列造成的效能問題。

2. **emwebhook 的防重複機制**：
   BEFORE_INSERT 會檢查 1 小時內是否已有相同 `emProgramIdV2 + contactListName` 的記錄，防止 Emarsys 重複觸發。

3. **emfiletask 的 IdType.NATIVE**：
   不自動產生 ID，由 Emarsys 回傳的檔案名稱作為唯一識別。

4. **emwebhook 分頁處理**：
   Contact List 可能包含數萬筆會員，因此透過 offset/limit 分頁，每次只取一頁呼叫 API，避免單一 function 超時。

5. **上傳目標差異**：
   - emcontacttask → Emarsys **WebDAV**（會員資料）
   - emsaletask / emproducttask / emstoretask → Emarsys **SFTP**（交易/商品/門市資料）
