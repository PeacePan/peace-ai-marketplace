# 點數系統腳本詳細說明

> 知識版本：`pointaccount` v50、`givepointsetting` v32、`importpoint` v29、`pointdiscount` v11、`feversocialpoints` v5、`pointpromotion` v15

## pointaccount 腳本目錄：`tables/memberRights/scripts/pointaccount/`

### `policy.ts` — 點數帳本新增驗證

**觸發方式**：Policy（新增/更新前）

**主要邏輯**：
- 只允許新增操作時驗證
- 驗證 lines.history（點數歷程表身）不得為空

**注意事項**：中台無法直接建立點數帳本，必須透過系統自動產生的歷程記錄來建立

---

### `change.ts` — 點數異動（call 函式）

**觸發方式**：CallFunction（API 呼叫）

**主要邏輯**：
- 接收 PointAccountChangeArgs 參數
- 透過 `MemberRights.pointChange` 執行點數異動

**注意事項**：參數為必填，未提供時拋出錯誤；實際點數邏輯由 MemberRights 統一管理

---

### `expiredPoint.ts` — 處理已過期點數

**觸發方式**：TodoJob（由 expiredPoint.cron.ts 建立）

**主要邏輯**：
- 驗證點數帳本有足夠餘額（localBalance > 0）且已過期（expiredAt <= 現在）
- 查詢對應會員是否存在（已封存則更新備註後返回）
- 呼叫 `MemberRights.pointChange` 扣除全部剩餘餘額

**輸出/副作用**：建立負向點數帳本記錄；若會員不存在則更新 sysMemo 為「已封存會員」

**注意事項**：完整錯誤處理，防止誤操作封存會員

---

### `expiredPoint.cron.ts` — 過期點數掃描排程

**觸發方式**：排程（FIVE_MINS，正式環境）

**主要邏輸**：
- 掃描過去一週內過期的點數帳本（localBalance > 0、expiredAt <= 現在、sysMemo 非「已封存會員」）
- 每次最多 5000 筆
- 建立待辦工作

**注意事項**：採滑動式掃描；防止重複掃描已處理紀錄

---

### `archivePointAccount.ts` — 封存指定點數帳本

**觸發方式**：TodoJob（由 archivePointAccount.cron.ts 建立）

**輸入條件**：
- 記錄符合特定條件（type=IMPORT、memo 含「完成寵物資料填寫 - 贈50點」、localBalance=50、建立時間 >= 2025-10-05）
- 或測試環境（isDev=true）

**主要邏輯**：
- 驗證符合封存條件
- 呼叫 archiveV2 API 封存帳本
- 建立待辦工作以重算會員剩餘點數

**注意事項**：防重複機制；只封存特定日期及內容的新會員贈點

---

### `archivePointAccount.cron.ts` — 點數帳本批次封存排程

**觸發方式**：排程（FIVE_MINS，正式環境）

**主要邏輯**：
- 查詢指定條件的點數帳本（2025-10-05 當天的新會員贈 50 點）
- 按建立時間排序
- 建立待辦工作

**注意事項**：限定特定日期範圍

---

### `recovery.ts` — 點數回補

**觸發方式**：Function（手動觸發）

**主要邏輯**：
- 若未提供 record，根據 memberName 與 maOrderId 查詢帳本
- 呼叫 `MemberRights.recoveryPoints` 計算並回補點數

**輸出/副作用**：建立回補點數的帳本記錄；回傳回補帳本編號

---

### `maRecoveryPoint.ts` — Magento 回補點數 API

**觸發方式**：CallFunction（Magento API 呼叫）

**輸入**：必填 memberName + magentoOrderId；可選 memo

**主要邏輯**：
- 驗證必填參數
- 查詢對應的點數帳本記錄
- 呼叫 `MemberRights.recoveryPoints` 執行回補

**應用場景**：Ma 訂單成立但未拋單倉庫時取消；Ma 訂單付款失敗時

---

### `deductPoint.ts` — 點數核銷 API

**觸發方式**：CallFunction（Magento / LocalPOS API 呼叫）

**輸入**：
- 必填：memberName、type（POINT_REDEMPTION 或 POINT_DISCOUNT）、usedPoints（> 0）
- Magento 呼叫時需 magentoOrderId
- LocalPOS 呼叫時需 sourceTable 與 sourceRecordName

**主要邏輯**：
- 完整驗證所有參數（類型、呼叫端身份、必填組合）
- 構建負向 PointAccountChangeArgs
- 呼叫 `MemberRights.pointChange` 執行核銷

**注意事項**：嚴格驗證呼叫端身份與參數搭配；支援兩種呼叫來源

---

## givepointsetting 腳本目錄：`tables/memberRights/scripts/givepointsetting/`

### `policy.ts` — 贈點活動設定驗證政策

**觸發方式**：Policy（新增/更新前）

**主要邏輯**（20+ 驗證規則）：
- 驗證時間邏輯（startAt < endAt、startAt > 現在時間）
- 驗證會員等級設定至少一種
- 驗證各等級的適用渠道不重複
- 驗證活動類型與渠道的合法組合
- 驗證表身配置與活動類型相符
- 驗證活動週期衝突（消費贈點、會員生日、寵物生日類型）
- 驗證點數有效期設定規則
- 驗證消費訂單滿額設定
- 自動產生禮物卡序號（若類型為 POINT_CARD 且序號為空）

**注意事項**：活動類型決定整個驗證路徑；防重複渠道設定

---

### `beforeUpdate.ts` — 贈點活動更新前保護

**觸發方式**：BeforeUpdate Hook

**主要邏輯**：
- 若現在時間已超過活動起日，觸發欄位保護機制
- 正式環境保護：startAt、expireDays、expiredAt、classicSetting/goldSetting/blackSetting 表身
- 非正式環境：只保護 expireDays、expiredAt（允許修改 startAt 以便測試）
- 嘗試修改受保護欄位時拋出錯誤，含欄位顯示名稱

**注意事項**：防止活動進行中修改規則，保護業務完整性

---

### `pointCardChange.ts` — 禮物卡點數變動

**觸發方式**：Function（手動觸發）

**主要邏輸入**：memberName、changeAmount、pointCardCode、channel 均必填

**主要邏輯**：驗證必填參數，呼叫 `MemberRights.pointCardChange`

---

### `usedChange.ts` — 贈點活動使用組數更新

**觸發方式**：Function（待辦工作觸發）

**主要邏輯**：
- 從待辦工作參數解析 allStoreUsedChange（全門市使用組數異動量）
- 計算新使用組數（舊值 + 異動值）
- 使用 ignorePolicy 旗標更新 allStoreUsed 欄位

---

### `changeStartAtForTest.ts` — 測試用活動起日變更

**觸發方式**：Function（測試環境限定）

**主要邏輯**：環境檢查（正式環境拋錯）+ 驗證參數 + 更新 startAt

**注意事項**：正式環境完全禁用

---

## importpoint 腳本目錄：`tables/memberRights/scripts/importpoint/`

### `policy.ts` — 匯入點數驗證政策

**觸發方式**：Policy（新增/更新時）

**主要邏輯**：
- 驗證點數有效期設定（expireDays 與 expiredAt 二擇一）
- 若非新增，驗證 processedLineId 為空才可修改會員名單（防止處理中修改）
- 當時間已過預約時間且任務狀態未執行/未完成/錯誤時，自動建立匯入待辦工作

**注意事項**：自動排程功能整合於政策中；防止處理中修改名單

---

### `importPoints.ts` — 執行批次匯入點數

**觸發方式**：TodoJob（由 policy 自動觸發或 importPoints.cron.ts）

**主要邏輯**：
- 驗證點數參數有效性
- 根據 processedLineId 計算待處理會員（斷點續傳）
- 每批 30 筆呼叫 `MemberRights.pointChange`
- 記錄 processedLineId 進度
- 全部完成則更新 taskStatus 為完成

**輸出/副作用**：建立批量點數帳本記錄；更新 processedLineId、taskStatus、taskError

**注意事項**：
- 時間限制：背景 40s、前景 20s
- 防重複：processedLineId 追蹤進度，支援斷點續傳

---

### `importPoints.cron.ts` — 點數匯入排程

**觸發方式**：排程（HOURLY）

**主要邏輯**：掃描預約時間已到且狀態未執行/錯誤/未完成的任務，建立待辦工作

**注意事項**：掃描限制當天 1 天範圍

---

### `cancel.ts` — 取消匯入任務

**觸發方式**：Function（手動觸發）

**主要邏輸入**：任務必須未開始執行（預約時間 > 現在）

**主要邏輯**：驗證時間條件，更新 taskStatus 為「已取消」

---

## pointdiscount 腳本目錄：`tables/memberRights/scripts/pointdiscount/`

### `policy.ts` — 點數折扣政策

**觸發方式**：Policy

**主要邏輯**：驗證時間邏輯（startAt < endAt）

---

## feversocialpoints 腳本目錄：`tables/memberRights/scripts/feversocialpoints/`

### `maDepositPoints.ts` — 社群平台點數歸戶（同步）

**觸發方式**：CallFunction（Magento API）

**主要邏輯**：
- 驗證：memberName、points（>0）、expiredAt、memo 均必填
- 查詢會員是否存在
- 建立 feversocialpoints 記錄
- 直接呼叫 pointChange 立即入帳（todoJobVersion=true）

**輸出/副作用**：建立 feversocialpoints 記錄；建立 pointaccount 記錄（sourceTable=feversocialpoints）

**注意事項**：同步實時入帳

---

### `maDepositPoints2.ts` — 社群點數歸戶（非同步事件版）

**觸發方式**：CallFunction（Magento API）

**主要邏輯**：
- 驗證參數
- 發送事件 `pubsub1/feversocialpoints/depositPoints/{memberName}`
- 不直接入帳，回傳模擬成功回應

**注意事項**：事件驅動版本，解耦 API 呼叫與入帳

---

### `maDepositPointsEvent.ts` — 社群點數歸戶事件監聽

**觸發方式**：EventService（監聽 pubsub1/feversocialpoints/depositPoints/* 事件）

**主要邏輯**：與 maDepositPoints.ts 完全相同，但接收事件物件而非 API 參數

**注意事項**：處理 maDepositPoints2 發送的事件

---

## pointpromotion 腳本目錄：`tables/memberRights/scripts/pointpromotion/`

### `policy.ts` — 點加金活動政策

**觸發方式**：Policy

**主要邏輯**：
- 驗證時間邏輯（startAt < endAt）
- 驗證每筆兌換品項數量邏輯（已兌換 <= 可兌換）
- 驗證品項單價必為整數
- 自動產生缺失的條碼（名稱+品項名）

**輸出/副作用**：返回去重後的驗證錯誤訊息集合；自動補齊條碼

---

### `calcRedeemableAmount.ts` — 計算已兌換數量

**觸發方式**：TodoJob（由 calcRedeemableAmount.cron.ts 建立）

**主要邏輯**：
- 驗證活動時間內、未達兌換上限
- 查詢允用門市清單
- 查詢活動期間內的銷售單（PAID/REFUND 狀態）
- 篩選指定活動名稱、門市、日期範圍的銷售單
- 彙整品項兌換數量（退款計負值）
- 更新各品項的 redeemedAmount

**輸出/副作用**：更新 redeemItems 表身的 redeemedAmount；更新 lastCalcSaleID

**注意事項**：單次限 10000 筆銷售單；lastCalcSaleID 防止重複計算；退款抵扣支援

---

### `calcRedeemableAmount.cron.ts` — 兌換數量計算排程

**觸發方式**：排程（HALF_HOURLY）

**主要邏輯**：掃描進行中的活動（startAt < 現在 < endAt），建立待辦工作
