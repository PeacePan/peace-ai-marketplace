# 優惠券系統腳本詳細說明

> 知識版本：`petparkcoupon` v92、`petparkcouponevent` v36、`petparkcouponassign` v25、`memberclass` v3、`memberautoarchive` v3、`membermerge` v21、`autoupdatememberclass` v9、`cresclabsyncmember` v15、`pospet` v4

## petparkcoupon 腳本目錄：`tables/memberRights/scripts/petparkcoupon/`

### `batchPolicy.ts` — 批次優惠券異動後標記會員待更新

**觸發方式**：BatchPolicy（批次編輯時自動觸發）

**主要邏輯**：
- 遍歷所有異動的優惠券記錄
- 偵測三種觸發情況：(1)新增非紙本優惠券、(2)優惠券被使用、(3)優惠券被返還
- 收集受影響的會員編號（使用 Map 去重）
- 為每個受影響會員建立「標記會員優惠券狀態」的待辦工作（queueNo=1）

**輸出/副作用**：建立 todoJobsV2 待辦工作記錄

**注意事項**：只在偵測到狀態異動時建立待辦，防止無謂觸發

---

### `beforeInsert.ts` — 新增優惠券前的券號產生與折價計算

**觸發方式**：BeforeInsert Hook

**主要邏輯**：
1. 隨機生成 8 位券號（4 位 16 進制 + 12 位時間戳）
2. 依據優惠券活動配置計算折價金額：
   - 固定金額折扣
   - 金額累贈（消費滿 X 贈 Y）
   - 百分比折扣（依符合篩選條件的商品金額計算）
3. 驗證不能建立已使用過的優惠券

**輸出/副作用**：填充 code 和 discount 欄位

**注意事項**：折扣金額使用 Math.floor()/Math.round() 確保整數；缺少來源資料時拋出錯誤

---

### `updateTypeValue.ts` — 補值優惠券 type 欄位

**觸發方式**：TodoJob（由 updateTypeValue.cron.ts 建立）

**主要邏輯**：
- 從 todoJob 參數讀取 queueNo（1-20）
- 查詢 type 為空的優惠券（limit 30000）
- 以 queueNo 對系統編號取模分散負載（20 個 worker）
- 每個 worker 最多更新 1500 筆

**輸出/副作用**：批量更新 type 欄位為 'ELECTRONIC'

**注意事項**：計時監控（console.time）；20 個 worker 避免衝突

---

### `updateTypeValue.cron.ts` — 補值 type 欄位排程

**觸發方式**：排程（TEN_MINS）

**主要邏輯**：
- 查詢是否存在 type 為空的優惠券
- 若存在，建立 20 筆待辦工作（queueNo 1-20）

---

### `updateDiscount.ts` — 更新優惠券折扣金額

**觸發方式**：Function（手動執行，傳入優惠券記錄）

**主要邏輯**：
- 查詢優惠券活動
- 若活動為折價券類型，重新計算折價值（使用 beforeInsert.ts 的計算邏輯）
- 若新舊折價不同，執行更新

**應用場景**：活動配置異動後重新計算既有優惠券的折扣金額

---

### `archiveDuplicate.ts` — 封存重複發放優惠券

**觸發方式**：TodoJob（手動執行）

**主要邏輯**：
- 從 startIndex（buffer）反向查詢該活動的重複發放記錄（每次 200 筆）
- 對每個會員的優惠券：
  - 未封存多於 1 筆：保留最新的，其餘封存
  - 全部已封存：反封存最新的 1 筆（確保會員有券可用）
- 為受影響會員建立「更新會員優惠券」待辦
- 返回 WORKING 並更新 buffer，支援分頁續執行

**輸出/副作用**：封存/反封存優惠券記錄；建立會員優惠券更新待辦

**注意事項**：目前硬編碼優惠券活動代碼 'PRICE_DISCOUNT00000436'；使用 startIndex 分頁

---

### `markMemberCouponStatus.ts` — 標記會員優惠券狀態為待更新

**觸發方式**：TodoJob（由 batchPolicy 或其他觸發器建立）

**主要邏輯**：
- 每次從 buffer 取出 100 個會員
- 查詢現有 couponStatus，若非「待更新」才建立「更新會員優惠券」待辦工作
- 若還有剩餘會員，返回 WORKING 更新 buffer；否則返回 DONE

**注意事項**：防重複（檢查狀態再決定是否建立待辦）；使用 writeConflictAutoRetry 裝飾器

---

### `maUseCoupon.ts` — 核銷優惠券

**觸發方式**：CallFunction（帶入 code、usedAt、usedSaleName）

**主要邏輯**：
1. 驗證必填參數
2. 查詢優惠券記錄
3. 驗證優惠券未被使用過
4. 查詢優惠券活動，依發放渠道判斷：
   - 門市/美容渠道：驗證使用時間在有效期內
   - 線上購：無有效期限制
5. 更新 usedAt 和 usedSaleName

**輸出/副作用**：更新優惠券使用狀態；回傳券號

---

### `maUnuseCoupon.ts` — 返還優惠券

**觸發方式**：CallFunction（帶入 memberName、code、usedSaleName）

**主要邏輯**：查詢已使用的優惠券，清除 usedAt 和 usedSaleName（設為 null）

**應用場景**：撤銷已核銷的優惠券（如退貨）

---

### `maIssueCoupon.ts` — 發放優惠券（主邏輯）

**觸發方式**：CallFunction（API 呼叫）

**輸入**：
- timing：'FIRST_SIGNIN_APP' | 'APPNEW_MEMBER' | 'EVENT_CODE'
- memberName：必填
- couponEventCode：当 timing='EVENT_CODE' 時必填

**主要邏輯**：
1. 驗證會員存在
2. 查詢會員現有優惠券對照表
3. 根據 timing 確定候選活動：
   - FIRST_SIGNIN_APP/APPNEW_MEMBER：查詢有效期內活動，排除已領取
   - EVENT_CODE：查詢活動代碼，判斷活動狀態（生效中/已結束/未開始/已領取）
4. 批量建立 petparkcoupon 記錄
5. 回傳券號陣列

**注意事項**：防重複機制（檢查是否已領取）；使用台灣時區判斷時間；使用 `calcCouponStartEndAt` 計算有效期

---

### `maIssueCoupon2.ts` — 發放優惠券（非同步事件版）

**觸發方式**：CallFunction（API 呼叫）

**主要邏輯**：發送事件 `pubsub1/petparkcoupon/maIssueCoupon/{memberName}`，不直接發放

**注意事項**：事件驅動版，解耦 API 與實際發放

---

### `maIssueCouponEvent.ts` — 發放優惠券（事件服務處理器）

**觸發方式**：EventService（監聽 pubsub1/petparkcoupon/maIssueCoupon/* 事件）

**主要邏輯**：與 maIssueCoupon.ts 完全相同，但接收事件物件

---

## petparkcouponevent 腳本目錄：`tables/memberRights/scripts/petparkcouponevent/`

### `policy.ts` — 優惠券活動設定驗證政策

**觸發方式**：Policy（新增或編輯優惠券活動時）

**主要邏輯**（超過 20 項驗證規則）：
- 活動起迄時間驗證（起日必早於迄日、新起日必晚於現在）
- 發放渠道與配置組合驗證（各渠道對應的篩選器限制）
- 優惠券類型與金額設定驗證（折價券/折扣券/免費券各自的規則）
- 累贈門檻與單筆消費門檻驗證
- 有效期設定驗證（各觸發時機對應不同的有效期要求）
- 優惠券活動代碼唯一性驗證（同時間不得重複）

**輸出/副作用**：回傳驗證錯誤訊息陣列（`\n` 分隔）

**注意事項**：涉及多表查詢（門市群組、門市配置）

---

### `beforeUpdate.ts` — 優惠券活動編輯保護

**觸發方式**：BeforeUpdate Hook

**主要邏輯**：
- 若活動起日已過期（現在時間 > 起日），禁止修改 13 個保護欄位
- 正式環境額外保護活動起日；非正式環境允許修改起日（方便測試）

**注意事項**：確保進行中的活動規則不被修改

---

### `changeStartAtForTest.ts` — 測試用活動起日調整

**觸發方式**：Function（測試環境限定）

**主要邏輯**：驗證非正式環境 + 驗證 changeStartAt 參數 + 直接更新 startAt（跳過 policy）

**注意事項**：正式環境拋出錯誤

---

## petparkcouponassign 腳本目錄：`tables/memberRights/scripts/petparkcouponassign/`

### `policy.ts` — 優惠券指定發放政策

**觸發方式**：Policy（新增或編輯時）

**主要邏輯**：
- 驗證優惠券活動編號必填
- 驗證活動觸發時機必須為「指定名單」
- 若 processedLineId 非空（已開始發放），禁止修改表身發放列表

**注意事項**：防止處理中途修改名單

---

### `assignCoupon.ts` — 批次發放指定名單優惠券

**觸發方式**：TodoJob（由 assignCoupon.cron.ts 建立）

**主要邏輯**：
1. 根據 processedLineId 篩選尚未發放的會員
2. 若無待發放會員，標記任務「已完成」並返回 DONE
3. 批量建立優惠券記錄（每 30 筆一批，使用 batchInsert）
4. 同時建立「標記會員優惠券狀態」待辦
5. 更新 processedLineId 和 cronStatus
6. 若超時，返回 WORKING 分頁繼續執行

**輸出/副作用**：批量建立 petparkcoupon 記錄；更新發放進度

**注意事項**：
- 時間限制：背景 40s、前景 20s
- 斷點續傳：processedLineId 追蹤進度
- 錯誤記錄：使用 dangerBatchUpdateV2 記錄 cronError

---

### `assignCoupon.cron.ts` — 指定發放排程

**觸發方式**：排程（FIVE_MINS）

**主要邏輯**：
- 查詢 issueAt 在前一天至今日間、cronStatus 為「未執行/未完成/錯誤」的任務
- 建立待辦工作（最多 2000 筆，優先處理「錯誤」狀態）

---

### `cancel.ts` — 取消指定發放

**觸發方式**：Function（手動執行）

**主要邏輯**：驗證發放時間未到，標記 cronStatus 為「已取消」

---

## memberclass 腳本目錄：`tables/memberRights/scripts/memberclass/`

### `policy.ts` — 會員等級設定驗證政策

**觸發方式**：Policy

**主要邏輯**（10 項驗證規則）：
- 起迄時間驗證
- 時間區間重疊檢查（查詢 memberclass 表確保無重疊）
- 金卡升級：最低消費金額與次數成對驗證
- 金卡續等：最低消費金額與次數成對驗證
- 黑卡升級：最低消費金額與次數成對驗證
- 黑卡續等：最低消費金額與次數成對驗證

---

## memberautoarchive 腳本目錄：`tables/memberRights/scripts/memberautoarchive/`

### `policy.ts` — 會員自動封存政策

**觸發方式**：Policy（新增時）

**主要邏輯**：
- 確認為新增操作
- 若會員尚未封存，直接呼叫 API 執行封存

**注意事項**：封存邏輯整合在 policy 中，新增即觸發

---

## membermerge 腳本目錄：`tables/memberRights/scripts/membermerge/`

### `policy.ts` — 會員合併政策

**觸發方式**：Policy（新增時）

**主要邏輯**：
- 驗證保留會員與封存會員編號不同
- 查詢兩個會員，確認沒有即將到期的會員

**注意事項**：防止會籍即將到期時執行合併

---

### `merge.ts` — 會員合併執行

**觸發方式**：TodoJob（由 merge.cron.ts 建立）

**主要邏輯（兩階段執行）**：

**第一階段（buffer.archived=false）**：
- 確認來源會員尚未封存
- 等待所有待更新等級的會員完成等級更新
- 遷移：pospet（寵物）、possale（銷售單）、posservicesale（服務銷售單）
- 轉移點數（pointChange）
- 標記執行中，返回 WORKING

**第二階段（buffer.archived=true）**：
- 比較兩會員的會籍歷程，選擇高等級或先開啟的資料
- 若今日已有相同等級的等級異動，不重複發放優惠券
- 更新保留會員的會籍歷程
- 封存來源會員
- 建立「已封存會員點數封存」待辦（archivePoint.ts）

**輸出/副作用**：更新寵物/銷售單會員資訊；轉移點數；更新會籍歷程；封存來源會員

**注意事項**：兩階段設計防止單次超時；合併排程限制時間 02:00-08:59（避免尖峰）

---

### `merge.cron.ts` — 會員合併排程

**觸發方式**：排程（FIVE_MINS，僅台灣時間 02:00-08:59）

**主要邏輸**：
- 防重複：若有進行中的待辦工作則跳過
- 查詢 taskStatus 為 null 或「錯誤」的合併任務
- 篩選保留/來源會員均已更新等級的任務
- 建立待辦工作

**注意事項**：時間限制避免尖峰時段；等級更新完成才可合併

---

### `archivePoint.ts` — 已封存會員點數封存

**觸發方式**：TodoJob（由 merge.ts 第二階段建立）

**主要邏輯**：
- 查詢封存會員的點數帳本（未封存的記錄）
- 每次封存 10 筆
- 若無剩餘未封存點數，更新會員 pointArchivedAt 欄位，返回 DONE
- 若有剩餘，更新 buffer 計數，返回 WORKING

**注意事項**：分頁執行，每次 10 筆

---

## autoupdatememberclass 腳本目錄：`tables/memberRights/scripts/autoupdatememberclass/`

### `policy.ts` — 自動更新等級排程設定政策

**觸發方式**：Policy（新增時）

**主要邏輯**：確認系統只有 1 筆排程記錄（若已存在則拒絕新增）

---

### `start.ts` — 開始自動更新等級

**觸發方式**：Function（手動執行）

**主要邏輯**：驗證狀態非「執行中/成功/失敗」，更新為「執行中」

---

### `cancel.ts` — 取消自動更新等級

**觸發方式**：Function（手動執行）

**主要邏輯**：驗證狀態非「取消」，更新為「取消」

---

### `updateDate.ts` — 推進會員等級計算日期

**觸發方式**：TodoJob（由 updateDate.cron.ts 建立）

**主要邏輯**：
- 驗證 updateClassAt 已設定
- 驗證狀態為「成功」（前一日所有會員已計算完成）
- updateClassAt 加 1 天，重設狀態為「執行中」

**注意事項**：用於排程的日期自動滾動

---

### `updateDate.cron.ts` — 日期推進排程

**觸發方式**：排程（TEN_MINS）

**主要邏輸**：掃描狀態為「成功」的排程檔，建立待辦工作執行 updateDate

---

## cresclabsyncmember 腳本目錄：`tables/memberRights/scripts/cresclabsyncmember/`

### `policy.ts` — 漸強同步任務政策

**觸發方式**：Policy（新增或編輯時，僅正式環境）

**主要邏輯**：
- 非正式環境不執行
- 狀態從其他→「已下載」時，建立「LINE 同步會員狀態」待辦
- 狀態從其他→「新建立」時，建立「取得漸強會員資料」待辦

**注意事項**：狀態轉移驅動工作流；非正式環境無效

---

### `appendMember.ts` — 從漸強下載會員 LINE 資料

**觸發方式**：TodoJob（由 policy 或 appendMember.cron.ts 建立）

**主要邏輯**：
- 呼叫漸強 API 以天為單位查詢會員 LINE 狀態更新
- 限制 6000 筆或 40 秒（防超時）
- 排除重複 LINE ID（保留更新時間較新的）
- 寫入 members 表身
- 若有分頁（nextToken），自動建立下一個任務接續

**輸出/副作用**：更新 cresclabsyncmember 的 members 表身

**注意事項**：nextToken 有效期 30 分鐘；分頁設計避免超時

---

### `appendMember.cron.ts` — 漸強資料下載排程

**觸發方式**：排程（HOURLY，僅 production 環境）

**主要邏輸**：掃描「新建立」狀態的任務，建立待辦工作；非正式環境返回禁用提示

---

### `syncMemberStatus.ts` — 同步 LINE 狀態至 posmember

**觸發方式**：TodoJob（由 policy 或 syncMemberStatus.cron.ts 建立）

**主要邏輯**：
- 每次從 members 表身取 100 個會員
- 以 emLineId 匹配 posmember
- 若狀態不同則更新 posmember.emLineIdStatus
- 標記同步完成；找不到的標記「已略過」
- 若有剩餘，返回 WORKING

**輸出/副作用**：更新 posmember.emLineIdStatus；標記 members 的 syncStatus

**注意事項**：分頁處理每次 100 筆；找不到的容錯標記

---

### `syncMemberStatus.cron.ts` — 狀態同步排程

**觸發方式**：排程（HOURLY，僅 production 環境）

**主要邏輯**：掃描「已下載」狀態的任務，建立待辦工作

---

### `createTask.cron.ts` — 建立漸強同步任務

**觸發方式**：排程（HOURLY，僅 production 環境）

**主要邏輸**：
- 若完全沒有同步任務，自動建立初始任務（日期設為 2022-11-18，漸強資料庫最早時間）
- 若最新任務已完成前一天數據，自動建立隔天任務
- 防止日期差距為 0 的重複建立

**注意事項**：自動啟動機制；日期自動推進
