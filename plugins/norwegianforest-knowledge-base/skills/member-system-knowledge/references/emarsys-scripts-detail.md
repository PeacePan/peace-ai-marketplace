# Emarsys CRM 同步腳本詳細說明

> 知識版本：`emcontacttask` v49、`emfiletask` v2、`emwebhook` v28、`emsaletask` v46、`emproducttask` v19、`emstoretask` v11

## emcontacttask 腳本目錄：`tables/emarsys/scripts/emcontacttask/`

### `genTask.cron.ts` — 定期產生會員匯入任務

**觸發方式**：排程（Cron2，定期執行）

**主要邏輯**：
- 掃描 emSyncStatus = '待同步' 的會員（_id > 上次任務中最後會員 _id）
- 執行時間限制 25 秒，每次最多 25,000 筆
- 每 5,000 筆會員建立一筆 emcontacttask 記錄
- 為新建任務自動建立「匯入會員資料」待辦工作（queueNo=1）

**輸出/副作用**：新增 emcontacttask 記錄（cronStatus='未執行'）；建立待辦工作

**注意事項**：排除虛擬會員 XXXX；以 _id 比對防止重複處理

---

### `genTask.dev.ts` — 開發環境測試用任務產生（每筆 1 會員）

**觸發方式**：isPublic（開發/staging 環境）

**主要邏輯**：查詢 3 筆待同步會員，每筆會員單獨建立一筆任務（cMemberSizePerTask=1）

**注意事項**：開發測試用，粒度最小

---

### `autoImport.ts` — 會員資料 CSV 產生與上傳至 WebDAV（兩階段）

**觸發方式**：TodoJob（「匯入會員資料」）

**主要邏輯**：

**第一階段（無 uploadedFileName buffer）**：
- 準備會員資料，建立 CSV（96 欄位）：
  - 會員基本資料（姓名、電話、生日、性別、地址等）
  - 優惠券資訊（到期天數分類）
  - 點數資訊（有效點數、效期）
  - 寵物資料
  - 等級資訊
  - 消費統計
- 特殊欄位處理：
  - MGM 推薦人需回查會員檔
  - 布林值轉 Y/N
  - 日期格式符合 Emarsys 規範
  - 字串清理（移除逗號、換行）
  - 寵物月份需額外計算
  - 部分欄位用 | 取代 ,
- 45 秒超時機制下上傳至 Emarsys WebDAV
- 更新 buffer 儲存檔名，狀態改 WORKING

**第二階段（有 uploadedFileName buffer）**：
- 更新任務 cronStatus 為「已上傳」
- 批量建立「更新會員同步回 Em 狀態」待辦工作（每 100 筆一個工作）

**輸出/副作用**：上傳 CSV 至 WebDAV；更新 emcontacttask 狀態；建立 updateEmSyncStatus 待辦

**注意事項**：使用 AbortController 超時控制；超時時狀態改「待重新上傳」等待重試排程

---

### `retry.cron.ts` — 重試失敗任務排程

**觸發方式**：排程（Cron2）

**主要邏輯**：掃描當天「待重新上傳」狀態的任務（limit=200），建立重試待辦工作

---

### `restartTask.ts` — 重置任務排程狀態

**觸發方式**：TodoJob

**主要邏輯**：
- 將任務 cronStatus 重置為「未執行」
- 重新建立「匯入會員資料」待辦工作

**應用場景**：人工干預重新執行

---

### `updateEmSyncStatus.ts` — 更新會員同步完成狀態

**觸發方式**：TodoJob（由 autoImport.ts 第二階段建立）

**主要邏輸入**：
- todoJob param：以逗號分隔的會員 _id 列表

**主要邏輯**：
- 批量更新 posmember 的 emSyncStatus = '已完成'（ignorePolicy）
- 檢查是否為任務中最後一筆，若是則更新 emcontacttask 的 cronStatus = '已完成'

**注意事項**：使用 writeConflictAutoRetry 包裝防止衝突

---

## emfiletask 腳本目錄：`tables/emarsys/scripts/emfiletask/`

### `scanFiles.cron.ts` — 掃瞄 WebDAV 新回傳檔案

**觸發方式**：排程（Cron2）

**主要邏輸**：
- 連接 Emarsys WebDAV，掃瞄 export 目錄內 `[CP]-*.csv` 格式的檔案
  - C 開頭 = 發放優惠券
  - P 開頭 = 發放點數
- 以時間戳重命名檔案（防止重複處理）
- 批量建立 emfiletask 記錄（status='PROCESSING'）

**注意事項**：掃瞄後立即重命名，防重複機制

---

### `processTasks.cron.ts` — 建立檔案處理待辦工作

**觸發方式**：排程（Cron2）

**主要邏輸**：
- 掃瞄 PROCESSING 狀態任務，或 24 小時內 FAILED 的任務
- 根據類型決定執行函數名稱（「發放點數」或「發放優惠券」）
- 分別建立待辦工作（queueNo: 優惠券=1，點數=2）

**注意事項**：優惠券優先執行（queueNo=1）

---

### `assignCoupon.ts` — 處理優惠券指定發放 CSV 檔

**觸發方式**：TodoJob（「發放優惠券」）

**主要邏輸**：
- 從 WebDAV 下載 CSV 檔
- 分批解析會員編號（每次 1,000 筆）
- 呼叫 assignCoupons() 建立 petparkcouponassign 記錄
- 更新任務進度（totalCount、processedCount）
- 全部完成時 status 改 COMPLETED，記錄 completedAt

**輸出/副作用**：建立 petparkcouponassign 記錄；更新 emfiletask 進度

**注意事項**：分批防超時；自動排除不存在的會員並附註 memo；寫入衝突自動重試

---

### `givePoint.ts` — 處理點數匯入 CSV 檔

**觸發方式**：TodoJob（「發放點數」）

**主要邏輸**：
- 從 WebDAV 下載 CSV 檔
- 解析檔名取得點數額度與到期日期/天數（支援 expireDays 或 expiredAt 兩種格式）
- 分批建立 importpoint 記錄
- 更新任務進度，全部完成時 status 改 COMPLETED

**輸出/副作用**：建立 importpoint 記錄；更新 emfiletask 進度

---

### `resetTask.ts` — 重置檔案任務

**觸發方式**：TodoJob

**主要邏輸**：清空所有進度欄位（totalCount、processedCount、completedAt、errorMessage、memo），status 改 PROCESSING

**應用場景**：人工重新執行

---

## emwebhook 腳本目錄：`tables/emarsys/scripts/emwebhook/`

### `policy.ts` — Webhook 記錄建立政策

**觸發方式**：Policy（新增時）

**主要邏輸**：
- 驗證參數格式：
  - ASSIGN_COUPON 類型需 couponEventName
  - GIVE_POINT 類型需 points 與到期設定
- 呼叫 Emarsys API 查詢 Contact List ID 與成員總數
- 更新 contactListId 和 contactListCount（ignorePolicy）

**注意事項**：驗證失敗直接拋錯；包含 Emarsys API 認證

---

### `beforeInsert.ts` — 防重複新增 Webhook 記錄

**觸發方式**：BeforeInsert Hook

**主要邏輯**：
- 查詢相同（behavior、emProgramIdV2、contactListName）的記錄
- 限制 1 小時內不可重複新增同配置

**注意事項**：1 小時窗口（Emarsys 可能設定重複執行，需要時間間隔）

---

### `createJobs.ts` — 建立 Webhook 執行任務工作

**觸發方式**：Function（手動觸發或由排程驅動）

**主要邏輯**：
1. 驗證任務狀態為「未執行」
2. 防重複檢查：查詢相同配置的進行中任務
   - 時間窗口 = 會員數 ÷ 10000 小時
   - 若在窗口內則跳過
3. 按會員總數分割任務：
   - 每工作處理 100 × 10 = 1,000 名會員
   - jobCount = ⌈contactListCount ÷ 1000⌉
4. 為每個工作建立待辦（queueNo: 優惠券=1，點數=2）
5. 更新 cronStatus = '執行中'

**輸出/副作用**：建立多筆待辦工作；更新 emwebhook 狀態

**注意事項**：防重複邏輯結合時間窗口與會員數；每工作包含 param:{offset, limit, count}

---

### `createJobs.cron.ts` — Webhook 任務建立排程

**觸發方式**：排程（Cron2）

**主要邏輸**：
1. 防重複：若已有執行中的「建立任務」待辦則跳過
2. 掃瞄 cronStatus=null 或「未執行」的 webhook
3. 按 (behavior, emProgramIdV2, contactListName) 分組
4. 第一筆直接建立；後續若在時間窗口內重疊則標記「不執行」
5. 批量更新被跳過記錄（cronStatus='不執行'，memo 記錄原因）
6. 批量建立「建立任務」待辦

**輸出/副作用**：更新 emwebhook 狀態；建立「建立任務」待辦工作

---

### `checkDone.ts` — 檢查 Webhook 任務完成狀態

**觸發方式**：Function（由 checkDone.cron.ts 建立的待辦工作驅動）

**主要邏輸**：
1. 驗證任務狀態為「執行中」
2. 查詢所有待辦工作與工作日誌，以 param 分組提取最新結果
3. 錯誤處理：
   - Write Conflict 或 timeout：自動重建任務重試
   - 其他錯誤：記錄為 cronError
4. 判定完成條件：無待辦工作且無錯誤
5. 完成後發送 Google Chat 通知（按 emProgramIdV2 排序）

**輸出/副作用**：更新 cronStatus（'已完成'/'執行中'/'錯誤'）；建立重試任務；發送 Google Chat

**注意事項**：智能重試機制（Write Conflict 自動重建）；錯誤時 mentionAll=true

---

### `checkDone.cron.ts` — 任務完成檢查排程

**觸發方式**：排程（Cron2）

**主要邏輸**：掃瞄「執行中」狀態的 webhook，建立「任務完成檢查」待辦；防重複

---

### `assignCoupons.ts` — 從 Contact List 發放優惠券

**觸發方式**：TodoJob（queueNo=1）

**主要邏輸**：
- 呼叫 Emarsys API 取得 Contact List 成員（按 offset/limit 分頁）
- 提取會員編號欄位（hardcode 欄位 ID=5901）
- 呼叫 assignCoupons() 建立 petparkcouponassign 記錄
- 發送 Google Chat 通知（包含 API 耗時、批次耗時、成功筆數）

**輸出/副作用**：建立 petparkcouponassign 記錄；發送 Google Chat

**注意事項**：Emarsys API 會員編號欄位 ID 固定為 5901，若更換需修改代碼

---

### `givePoints.ts` — 從 Contact List 贈送點數

**觸發方式**：TodoJob（queueNo=2）

**主要邏輸**：
- 呼叫 Emarsys API 取得 Contact List 成員
- 提取會員編號
- 呼叫 givePoints() 建立 importpoint 記錄
- 發送 Google Chat 通知

**注意事項**：邏輯同 assignCoupons，僅執行函數不同

---

## emsaletask 腳本目錄：`tables/emarsys/scripts/emsaletask/`

### `policy.ts` — 銷售任務建立政策

**觸發方式**：Policy（新增時）

**主要邏輸**：
- 若為手動新增（cronStatus='MANUAL' 或空）
- 呼叫 `Emarsys.findEmarsysSaleTaskLineRows()` 查詢待匯出銷售資料
- 自動補齊表身：posSaleItems、posReturnItems、posServiceSaleItems

**注意事項**：手動新增無須手填表身，policy 自動補齊

---

### `autoImport.ts` — 銷售資料自動匯出至 SFTP

**觸發方式**：Function（由待辦工作或手動觸發）

**主要邏輯**：
- 狀態為 DONE 或 MANUAL 則中止
- 呼叫 `Emarsys.getEmarsysSaleCSV()` 產生 CSV
- 上傳至 SFTP：`Emarsys.uploadCSV()`
- 更新 uploadedFile、cronStatus='DONE'、cronLog（含各階段執行時間）

**注意事項**：writeConflictAutoRetry 防衝突；完整執行日誌記錄

---

### `autoImport.cron.ts` — 銷售資料匯出排程

**觸發方式**：排程（HALF_HOURLY）

**主要邏輸**：
- 呼叫 `Emarsys.findEmarsysSaleTaskLineRows()` 查詢待匯出資料
- 若無資料則不建立任務
- 新增 emsaletask 記錄（cronStatus='NEW'，policy 自動補齊表身）
- 建立「自動匯出銷售資料」待辦工作

---

### `fallback.ts` — 銷售資料重新匯出（針對已完成任務）

**觸發方式**：Function（手動執行，需 MANUAL 狀態）

**主要邏輸**：
- 驗證 cronStatus = 'MANUAL'
- 使用 isFallback=true 重新產生 CSV 並上傳 SFTP
- 只更新 uploadedFile（不更新 cronStatus、cronLog）

**應用場景**：已完成任務但需要重新上傳

---

### `manualImport.ts` — 銷售資料手動匯出

**觸發方式**：Function（手動執行，需 MANUAL 狀態）

**主要邏輸**：同 fallback.ts，僅 isFallback 標記差異

---

### `_type.ts` — Emarsys 銷售資料型別定義

**包含**：
- `EmarsysSalesDataItemPart`：商品層級型別（item、price、quantity、campaign 標籤等）
- `EmarsysSalesBodyPart`：訂單層級型別（order、timestamp、customer、channel、store、coupon、campaign 等）

---

## emproducttask 腳本目錄：`tables/emarsys/scripts/emproducttask/`

### `policy.ts` — 商品任務建立政策

**觸發方式**：Policy（新增時）

**主要邏輯**：
- 建立 EmarsysCSV 檔案類別（若不存在）
- 建立或重置「Emarsys商品任務暫存檔」（file2 記錄）
- 查詢 item 表最新 _id 作為逆向掃描起點
- 更新 taskTable='item'、lastId、cronStatus='未執行'

---

### `autoImport.ts` — 商品資料批次匯出（分兩階段）

**觸發方式**：Function（由 autoImport.cron.ts 建立的待辦工作）

**主要邏輯**：

**狀態為「待上傳」**：
- 從 FILE2 下載 CSV，上傳至 SFTP，標記完成

**其他狀態**：
- 準備對照表（品牌、分類、動物種類）
- 15 秒內逆向掃描（按 _id <= lastId，每次 200 筆）：
  - item 表掃完後切換至 serviceitem 表
  - 最後固定附加 1 筆包月服務料件
- CSV 10 欄位：商品名、品牌、價格、分類、寵物、大小、敘述、年齡、標籤等
- 上傳至 FILE2 暫存，記錄檔案大小
- 完成時改狀態「待上傳」

**注意事項**：
- 商品品名僅 1 字需重複（如「狗」→「狗狗」）
- CSV 防重複：記錄已存在商品編號
- 字串清理（逗號、雙引號轉義）
- 日期格式化

---

### `autoImport.cron.ts` — 商品匯出執行排程

**觸發方式**：排程（Cron2，DAILY）

**主要邏輸**：掃描未完成任務（取最舊的 1 筆），建立「執行匯入商品資料」待辦

---

### `insertTask.cron.ts` — 商品任務建立排程

**觸發方式**：排程（Cron2，DAILY）

**主要邏輸**：
- 確認沒有未完成任務
- 若無，新建 emproducttask（policy 自動補齊欄位）
- 建立待辦工作

**注意事項**：一日一筆；有未完成任務則複用

---

## emstoretask 腳本目錄：`tables/emarsys/scripts/emstoretask/`

### `autoImport.ts` — 門市資料 CSV 產生與上傳至 SFTP

**觸發方式**：Function（由待辦工作驅動）

**主要邏輸**：
- 掃描所有 posstore 記錄（limit=2,000，按門市名稱排序）
- 產生 15 欄位 CSV：
  - store_id、name、city、address、座標、營運面積、物流連結、郵編
  - 分類（| 分隔）、開店日期、狀態（OPEN→開店、CLOSE→閉店）
  - country code 固定 886（台灣）
- SFTP 連接與上傳（含重試，retries=2）
- 更新 uploadedFile、cronStatus='已完成'、cronError=null

**注意事項**：日期格式：openDate 用 YYYY-MM-DD，其他用 ISO 8601；記錄 SFTP 操作日誌

---

### `autoImport.cron.ts` — 門市資料匯出排程

**觸發方式**：排程（HOURLY）

**主要邏輸**：
- 檢查台灣時間當日是否已有任務
- 若無，新建 emstoretask（cronStatus='未執行'）
- 建立「執行自動匯入門市資料」待辦

**注意事項**：一日一筆（台灣時區檢查）

---

### `reset.ts` — 重置門市任務

**觸發方式**：Function（手動執行）

**主要邏輸**：清空 uploadedFile、cronError、sftpLog，重置 cronStatus='未執行'

**應用場景**：人工重新執行

---

## Emarsys 系統關鍵設計模式

### 上傳目標區分
| 表格 | 上傳目標 | 格式 | 說明 |
|------|---------|------|------|
| emcontacttask | WebDAV | CSV（96 欄） | 會員資料雙向通訊（上傳後等待回傳檔案） |
| emfiletask | 讀取 WebDAV | CSV | 掃瞄 Emarsys 回傳的 C/P 開頭檔案 |
| emsaletask | SFTP | CSV | 銷售/退貨資料 |
| emproducttask | SFTP（暫存 FILE2 後） | CSV（10 欄） | 商品資料，逆向掃描 |
| emstoretask | SFTP | CSV（15 欄） | 門市資料，一次全量 |

### 防重複/防超時設計
- **emcontacttask**：使用 AbortController 45 秒超時；超時後狀態改「待重新上傳」交給 retry.cron 重試
- **emwebhook**：使用時間窗口（會員數 ÷ 10000 小時）防止重複發放
- **emfiletask**：掃瞄後立即重命名檔案，防止重複下載
- **emproducttask**：兩階段設計（暫存到 FILE2 → 再上傳 SFTP），確保大量資料不超時

### Google Chat 通知
- **emwebhook**（checkDone.ts、assignCoupons.ts、givePoints.ts）：每次完成任務或發生錯誤時通知
- 錯誤時 mentionAll=true，確保相關人員收到通知
- 通知包含環境標籤（DEV/STAGING/PROD）與執行耗時
