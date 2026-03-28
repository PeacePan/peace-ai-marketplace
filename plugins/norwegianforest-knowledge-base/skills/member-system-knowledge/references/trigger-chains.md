# 會員系統表格觸發鏈

> 依照 `norwegianforest-engineering-mindset:tracking-redundant-policy-triggers` SKILL 的方法論繪製。
>
> **關鍵原則**：`ignorePolicy: true` 只跳過 `policy` 和 `batchPolicy`，**不跳過** `beforeInsert`、`beforeUpdate`、`batchBeforeInsert`、`batchBeforeUpdate` 這四個 before hook。

---

## 觸發鏈索引

1. [結帳後贈點流程](#1-結帳後贈點流程)
2. [結帳後發券流程](#2-結帳後發券流程)
3. [會員等級更新流程](#3-會員等級更新流程)
4. [等級異動發放流程](#4-等級異動發放流程)
5. [點數欄位更新流程](#5-點數欄位更新流程)
6. [優惠券欄位更新流程](#6-優惠券欄位更新流程)
7. [生日月發放流程](#7-生日月發放流程)
8. [Emarsys 會員同步流程（推送）](#8-emarsys-會員同步流程推送)
9. [Emarsys Webhook 回調流程（回推）](#9-emarsys-webhook-回調流程回推)
10. [emfiletask 回傳檔案處理流程](#10-emfiletask-回傳檔案處理流程)
11. [優惠券指定發放流程](#11-優惠券指定發放流程)
12. [批次匯入點數流程](#12-批次匯入點數流程)
13. [FeverSocial 點數歸戶流程](#13-feversocial-點數歸戶流程)
14. [會員合併流程](#14-會員合併流程)
15. [會員自動封存流程](#15-會員自動封存流程)
16. [CrescLab LINE 狀態同步流程](#16-cresclab-line-狀態同步流程)

---

## 1. 結帳後贈點流程

```
[possale 完成結帳]
  │
  └─ FIVE_MINS cron → issuePoints.cron.ts
        │
        └─ 建立 posmember.issuePoints TodoJob
              │
              └─ [issuePoints.ts] 執行
                    │
                    ├─ 寫入 possale/posservicesale/mashoporder/mapartialorder
                    │   更新 issuePointStatus = 完成/錯誤 [ignorePolicy: true]
                    │   → 對應表格的 batchBeforeUpdate 仍然觸發！
                    │
                    └─ 透過 MemberRights.pointChange 寫入 pointaccount [INSERT]
                          │
                          └─ [pointaccount.policy] 驗證歷程表身非空
                          │
                          └─ 更新 posmember.pointsStatus = '待更新' [ignorePolicy: 未設定]
                                └─ [posmember.policy] 執行
                                      └─ 偵測 emSyncStatus 欄位異動
                                            → 可能設定 emSyncStatus = '待同步'
```

**注意**：issuePoints 每次最多處理 20 張銷售單，防止單次執行超時。

---

## 2. 結帳後發券流程

```
[possale 完成結帳]
  │
  └─ FIVE_MINS cron → issueConsumptionCoupon.cron.ts
        │
        └─ 建立 posmember.issueConsumptionCoupon TodoJob
              │
              └─ [issueConsumptionCoupon.ts] 執行
                    │
                    ├─ 寫入 possale/posservicesale/mashoporder
                    │   更新優惠券發放狀態 [ignorePolicy: 未設定]
                    │   → 對應表格的 policy/batchPolicy 皆觸發
                    │
                    └─ 寫入 petparkcoupon [INSERT，ignorePolicy: 未設定]
                          │
                          ├─ [petparkcoupon.beforeInsert] 執行（always runs）
                          │   → 生成 8 位券號
                          │   → 計算折價金額（固定/累贈/百分比）
                          │
                          └─ [petparkcoupon.batchPolicy] 執行
                                │
                                ├─ 偵測「新增非紙本優惠券」條件
                                │
                                └─ 建立 markMemberCouponStatus TodoJob [queueNo=1]
                                      │
                                      └─ [markMemberCouponStatus.ts] 執行
                                            │
                                            └─ 若 couponStatus ≠ '待更新'，建立 updateCoupons TodoJob
                                                  │
                                                  └─ [updateCoupons.ts] 更新 posmember 優惠券欄位
```

**注意**：issueConsumptionCoupon 每次最多處理 10 張銷售單。

---

## 3. 會員等級更新流程

```
[updateMemberClass.cron.ts] 掃描昨日有消費/退款/會籍到期/合併影響的會員
  │
  └─ 建立 posmember.updateMemberClass TodoJob
        │
        └─ [updateMemberClass.ts] 執行
              │
              ├─ 計算會籍累計消費金額與次數
              │
              ├─ 寫入 possale/posservicesale 等 8 種銷售表
              │   更新 memberClassStatus = 完成 [ignorePolicy: 未設定]
              │   → 各表格的 policy/batchPolicy 皆觸發
              │
              └─ 寫入 posmember [ignorePolicy: 未設定]
                    │
                    ├─ 更新 memberClass/memberStartAt/memberEndAt
                    │
                    └─ [posmember.policy] 執行
                          │
                          └─ 偵測會籍歷程最後一筆異動
                                │
                                ├─ 重算 consumerlifeTotal
                                │
                                ├─ 觸發 issueClassChangeCoupon TodoJob ──→ [見流程 4]
                                │
                                └─ 觸發 issueClassChangePoints TodoJob ──→ [見流程 4]
```

---

## 4. 等級異動發放流程

```
[posmember.policy 觸發] 或 [issueClassChangeCoupon.cron.ts 補跑]
  │
  ├─ [issueClassChangeCoupon.ts]
  │     │
  │     ├─ 只處理系統/Magento 自動建立的會籍歷程
  │     │   (允許清單：/{/、/更新會員等級函式/、/^token:Magento/)
  │     │
  │     ├─ 寫入 petparkcoupon [INSERT，ignorePolicy: 未設定]
  │     │   → [petparkcoupon.beforeInsert] 執行（生成券號）
  │     │   → [petparkcoupon.batchPolicy] 執行（觸發 markMemberCouponStatus）
  │     │
  │     └─ 寫入 posmember（更新會籍歷程的 issuedCouponStatus）[ignorePolicy: true]
  │           → [posmember.beforeUpdate] 仍然觸發（before hook！）
  │           → [posmember.policy] 被跳過
  │
  └─ [issueClassChangePoints.ts]
        │
        ├─ 透過 MemberRights.pointChange 寫入 pointaccount
        │   → [pointaccount.policy] 驗證
        │   → 更新 posmember.pointsStatus = '待更新'
        │       → [posmember.policy] 執行
        │
        └─ 寫入 posmember（更新會籍歷程的 issuedPointStatus）[ignorePolicy: true]
              → [posmember.beforeUpdate] 仍然觸發（before hook！）
              → [posmember.policy] 被跳過
```

---

## 5. 點數欄位更新流程

```
[posmember.pointsStatus = '待更新'] 的設定來源：
  - issuePoints.ts 發點後
  - issueClassChangePoints.ts 發點後
  - importPoints.ts 匯入後
  - feversocialpoints/maDepositPoints.ts 歸戶後
  - recalcPointBalance.ts 修正後
  │
  └─ FIVE_MINS cron → updatePoints.cron.ts
        │
        └─ 建立 posmember.updatePoints TodoJob
              │
              └─ [updatePoints.ts] 執行
                    │
                    ├─ 讀取 pointaccount 最新餘額
                    │
                    └─ 寫入 posmember
                          更新 pointsValid/pointsEndTime/pointsTotal [ignorePolicy: 未設定]
                          同時設定 emSyncStatus = '待同步'
                          │
                          └─ [posmember.policy] 執行
                                └─ 偵測 emSyncStatus 欄位異動 → 確認待同步狀態
```

---

## 6. 優惠券欄位更新流程

```
[posmember.couponStatus = '待更新'] 的設定來源：
  - petparkcoupon INSERT/使用/返還後，透過 batchPolicy → markMemberCouponStatus
  │
  └─ HOURLY cron → updateCoupons.cron.ts
        │
        └─ 建立 posmember.updateCoupons TodoJob
              │
              └─ [updateCoupons.ts] 執行
                    │
                    └─ 讀取 petparkcoupon 記錄
                          計算到期天數分類、數量
                          寫入 posmember 優惠券相關欄位 [ignorePolicy: 未設定]
                          │
                          └─ [posmember.policy] 執行
```

---

## 7. 生日月發放流程

```
每月 1 號 04:00-22:00 → issueBirthMonthCouponAndPoint.cron.ts
  │
  └─ 篩選生日月符合的會員（排除昨天有消費/退款的）
        │
        └─ 建立 posmember.issueBirthMonthCouponAndPoint TodoJob
              │
              └─ [issueBirthMonthCouponAndPoint.ts] 執行
                    │
                    ├─ 寫入 petparkcoupon [INSERT，ignorePolicy: 未設定]
                    │   → [petparkcoupon.beforeInsert] 執行
                    │   → [petparkcoupon.batchPolicy] → markMemberCouponStatus TodoJob
                    │
                    ├─ 透過 MemberRights.pointChange 寫入 pointaccount
                    │   → 更新 posmember.pointsStatus = '待更新'
                    │
                    └─ 寫入 posmember
                          更新 issueBirthMonthPointAt（防重複發放標記）[ignorePolicy: 未設定]
                          → [posmember.policy] 執行
```

---

## 8. Emarsys 會員同步流程（推送）

```
[posmember.emSyncStatus = '待同步'] 的設定來源：
  - posmember.policy 偵測到 EM 同步欄位異動
  - posmember.beforeUpdate.ts 偵測到 emLineIdStatus 異動
  - updatePoints.ts 發點更新後
  - emUpdateMemberAfterFinishCheckout.ts 結帳後
  │
  └─ Cron → emcontacttask.genTask.cron.ts
        │
        └─ 建立 emcontacttask（每 5000 名會員一筆）
              │
              └─ [emcontacttask.autoImport.ts] 第一階段
                    │
                    ├─ 準備 96 欄位 CSV（讀取多表資料）
                    ├─ 上傳至 Emarsys WebDAV（45 秒超時機制）
                    │   超時 → cronStatus = '待重新上傳' → retry.cron.ts 重試
                    │
                    └─ [emcontacttask.autoImport.ts] 第二階段
                          │
                          └─ 建立 updateEmSyncStatus TodoJob（每 100 名會員一個）
                                │
                                └─ [updateEmSyncStatus.ts] 執行
                                      │
                                      ├─ 寫入 posmember.emSyncStatus = '已完成'
                                      │   [ignorePolicy: true]
                                      │   → [posmember.beforeUpdate] 仍然觸發！
                                      │       檢查手機號碼重複 + LINE ID 狀態
                                      │   → [posmember.policy] 被跳過
                                      │
                                      └─ 若為任務最後一批 → 更新 emcontacttask.cronStatus = '已完成'
```

---

## 9. Emarsys Webhook 回調流程（回推）

```
[Emarsys 呼叫 Webhook 端點] → emwebhook INSERT
  │
  ├─ [emwebhook.beforeInsert] 執行（always runs）
  │   → 1 小時內相同配置不得重複新增（防重複）
  │
  └─ [emwebhook.policy] 執行
        │
        ├─ 驗證參數格式
        └─ 呼叫 Emarsys API 取得 contactListId 與 contactListCount
              │
              └─ 寫入 emwebhook（更新 contactListId/Count）[ignorePolicy: true]

[emwebhook.createJobs.cron.ts] 排程掃描
  │
  └─ 建立「建立任務」TodoJob
        │
        └─ [createJobs.ts] 執行
              │
              ├─ 防重複：時間窗口 = 會員數 ÷ 10000 小時
              │
              ├─ 建立 assignCoupons TodoJob（ASSIGN_COUPON 類型，queueNo=1）
              │   │
              │   └─ [assignCoupons.ts] 呼叫 Emarsys API 取 Contact List 成員
              │         │
              │         └─ 建立 petparkcouponassign [INSERT] ──→ [見流程 11]
              │
              └─ 建立 givePoints TodoJob（GIVE_POINT 類型，queueNo=2）
                    │
                    └─ [givePoints.ts] 呼叫 Emarsys API 取 Contact List 成員
                          │
                          └─ 建立 importpoint [INSERT] ──→ [見流程 12]

[emwebhook.checkDone.cron.ts] 定期檢查完成狀態
  │
  └─ [checkDone.ts] 執行
        │
        ├─ 若有 Write Conflict/timeout → 自動重建任務重試
        ├─ 若全部完成 → cronStatus = '已完成'
        └─ 發送 Google Chat 通知
```

---

## 10. emfiletask 回傳檔案處理流程

```
[Emarsys WebDAV export 目錄] 出現 C-*.csv 或 P-*.csv 檔案
  │
  └─ [emfiletask.scanFiles.cron.ts] 掃描並重命名（防重複）
        │
        └─ 建立 emfiletask（C=發放優惠券，P=發放點數）
              │
              └─ [emfiletask.processTasks.cron.ts] 建立待辦工作
                    │
                    ├─ C 類型 → 「發放優惠券」TodoJob（queueNo=1）
                    │   │
                    │   └─ [assignCoupon.ts] 下載 CSV，分批建立 petparkcouponassign
                    │         ──→ [見流程 11]
                    │
                    └─ P 類型 → 「發放點數」TodoJob（queueNo=2）
                          │
                          └─ [givePoint.ts] 下載 CSV，分批建立 importpoint
                                ──→ [見流程 12]
```

---

## 11. 優惠券指定發放流程

```
[petparkcouponassign INSERT] 的來源：
  - emwebhook.assignCoupons.ts
  - emfiletask.assignCoupon.ts
  - 手動建立
  │
  └─ [petparkcouponassign.policy] 執行
        │
        └─ 驗證活動 timing = '指定名單'；processedLineId 為空才可修改名單

[petparkcouponassign.assignCoupon.cron.ts] 排程掃描
  │
  └─ 建立 petparkcouponassign.assignCoupon TodoJob
        │
        └─ [assignCoupon.ts] 執行（斷點續傳 processedLineId）
              │
              ├─ 批次建立 petparkcoupon（每 30 筆，batchInsert）[ignorePolicy: 未設定]
              │   │
              │   ├─ [petparkcoupon.batchBeforeInsert] 執行（若存在）
              │   │
              │   └─ [petparkcoupon.batchPolicy] 執行
              │         └─ 建立 markMemberCouponStatus TodoJob
              │               └─ 更新 posmember.couponStatus = '待更新'
              │                     └─ HOURLY cron → updateCoupons.ts
              │
              └─ 建立 markMemberCouponStatus TodoJob（確保優惠券欄位更新）
```

---

## 12. 批次匯入點數流程

```
[importpoint INSERT] 的來源：
  - emwebhook.givePoints.ts
  - emfiletask.givePoint.ts
  - 手動建立
  │
  └─ [importpoint.policy] 執行
        │
        └─ 驗證到期設定；若時間已到且狀態異常 → 自動建立 importPoints TodoJob

[importpoint.importPoints.cron.ts] 排程掃描（HOURLY）
  │
  └─ 建立 importpoint.importPoints TodoJob
        │
        └─ [importPoints.ts] 執行（斷點續傳 processedLineId，每批 30 筆）
              │
              └─ 透過 MemberRights.pointChange 批次寫入 pointaccount
                    │
                    └─ [pointaccount.policy] 驗證
                    │
                    └─ 更新 posmember.pointsStatus = '待更新'
                          └─ FIVE_MINS cron → updatePoints.ts ──→ [見流程 5]
```

---

## 13. FeverSocial 點數歸戶流程

```
[外部 API 呼叫 maDepositPoints / maDepositPoints2]
  │
  ├─ maDepositPoints（同步）：
  │   ├─ 建立 feversocialpoints [INSERT，ignorePolicy: 未設定]
  │   │   → [feversocialpoints.policy] 執行（若存在）
  │   │
  │   └─ 直接呼叫 MemberRights.pointChange → 寫入 pointaccount
  │         → 更新 posmember.pointsStatus = '待更新'
  │         → FIVE_MINS cron → updatePoints.ts ──→ [見流程 5]
  │
  └─ maDepositPoints2（非同步事件版）：
        └─ 發送事件 pubsub1/feversocialpoints/depositPoints/{memberName}
              │
              └─ [maDepositPointsEvent.ts] 接收事件
                    └─ 邏輯同 maDepositPoints
```

---

## 14. 會員合併流程

```
[membermerge INSERT] → [membermerge.policy] 驗證
  └─ 驗證兩會員不同，且不在即將到期階段

[membermerge.merge.cron.ts] 排程（台灣時間 02:00-08:59）
  │
  └─ 建立 membermerge.merge TodoJob（onJobRetry: 10）
        │
        └─ [merge.ts] 第一階段（buffer.archived=false）
              │
              ├─ 寫入 pospet.memberName [batchUpdateV2，ignorePolicy: true]
              │   → [pospet.batchBeforeUpdate] 仍然觸發！（before hook）
              │   → [pospet.batchPolicy] 被跳過
              │
              ├─ 寫入 possale.memberName [batchUpdateV2，ignorePolicy: true]
              │   → possale 的 batchBeforeUpdate 仍然觸發！
              │
              ├─ 寫入 posservicesale.memberName [batchUpdateV2，ignorePolicy: true]
              │   → posservicesale 的 batchBeforeUpdate 仍然觸發！
              │
              ├─ 透過 MemberRights.pointChange 轉移點數 → 寫入 pointaccount
              │   → [pointaccount.policy] 執行
              │   → 更新 posmember.pointsStatus = '待更新'
              │
              └─ [merge.ts] 第二階段（buffer.archived=true）
                    │
                    ├─ 封存來源會員 posmember [ignorePolicy: 未設定]
                    │   → [posmember.policy] 執行
                    │   → [posmember.beforeUpdate] 執行
                    │
                    ├─ 更新保留會員 posmember 的會籍歷程 [ignorePolicy: 未設定]
                    │   → [posmember.policy] 執行（重算消費金額等）
                    │
                    └─ 建立 archivePoint TodoJob
                          │
                          └─ [archivePoint.ts] 逐批封存 pointaccount（每批 10 筆）
                                → 更新 posmember.pointArchivedAt
```

**注意**：合併時 pospet/possale/posservicesale 雖設 ignorePolicy:true，其 batchBeforeUpdate hook 仍觸發，需注意這些表格的 batchBeforeUpdate 是否有副作用。

---

## 15. 會員自動封存流程

```
[memberautoarchive INSERT]
  │
  └─ [memberautoarchive.policy] 執行
        │
        └─ 若 posmember 尚未封存 → 立即呼叫 archive API 封存 posmember
              │
              └─ [posmember.policy] 執行（封存觸發的異動）
              └─ [posmember.beforeUpdate] 執行
```

---

## 16. CrescLab LINE 狀態同步流程

```
HOURLY cron → cresclabsyncmember.createTask.cron.ts（production only）
  │
  └─ 建立 cresclabsyncmember（status='新建立'）
        │
        └─ [cresclabsyncmember.policy] 偵測 status='新建立'
              └─ 建立 appendMember TodoJob

[appendMember.ts] 執行
  │
  └─ 呼叫漸強 API 下載 LINE 狀態（最多 6000 筆，40 秒超時）
        │
        └─ 寫入 members 表身 + 更新 status='已下載' [ignorePolicy: 未設定]
              │
              └─ [cresclabsyncmember.policy] 偵測 status='已下載'
                    └─ 建立 syncMemberStatus TodoJob

[syncMemberStatus.ts] 執行（每次 100 筆）
  │
  └─ 寫入 posmember.emLineIdStatus [batchUpdateV2，ignorePolicy: true]
        │
        ├─ [posmember.beforeUpdate] 仍然觸發！（before hook）
        │   → 偵測 emLineIdStatus 異動
        │   → 若 emSync=是，設定 emSyncStatus = '待同步'
        │   → [posmember.policy] 被跳過（ignorePolicy:true）
        │
        └─ emSyncStatus='待同步' 設定後
              └─ Cron → emcontacttask.genTask.cron.ts ──→ [見流程 8]
```

**關鍵**：`syncMemberStatus.ts` 寫 `posmember` 雖設 `ignorePolicy: true` 跳過了 policy，但 `beforeUpdate.ts` 仍會執行，並在 emLineIdStatus 改變時將 emSyncStatus 設為「待同步」，從而觸發整個 Emarsys 同步鏈。

---

## 觸發鏈總覽（ignorePolicy 關鍵路徑）

| 寫入操作 | ignorePolicy | policy 是否觸發 | before hook 是否觸發 | 需注意的副作用 |
|---------|-------------|---------------|-------------------|-------------|
| issuePoints → possale 更新 | true | ❌ 跳過 | ✅ batchBeforeUpdate 仍觸發 | possale.batchBeforeUpdate 的邏輯 |
| issueClassChangeCoupon → posmember 更新 | true | ❌ 跳過 | ✅ beforeUpdate 仍觸發 | 手機號碼重複檢查、LINE ID 狀態偵測 |
| issueClassChangePoints → posmember 更新 | true | ❌ 跳過 | ✅ beforeUpdate 仍觸發 | 同上 |
| merge → pospet/possale/posservicesale 更新 | true | ❌ 跳過 | ✅ batchBeforeUpdate 仍觸發 | 各表格 batchBeforeUpdate 的副作用 |
| syncMemberStatus → posmember 更新 | true | ❌ 跳過 | ✅ beforeUpdate 仍觸發 | **設定 emSyncStatus='待同步'，觸發完整 Emarsys 同步鏈** |
| updateEmSyncStatus → posmember 更新 | true | ❌ 跳過 | ✅ beforeUpdate 仍觸發 | 手機號碼重複檢查（通常無副作用） |
| cresclabsyncmember.policy → emwebhook 更新 | true | ❌ 跳過 | ✅ beforeUpdate 仍觸發 | emwebhook.beforeUpdate（若存在）的邏輯 |
| issueConsumptionCoupon → petparkcoupon INSERT | 未設定 | ✅ 觸發 | ✅ 觸發 | batchPolicy → markMemberCouponStatus |
| assignCoupon → petparkcoupon batchInsert | 未設定 | ✅ 觸發 | ✅ 觸發 | batchPolicy → markMemberCouponStatus |
| importPoints → pointaccount INSERT | N/A（透過 MemberRights） | ✅ 觸發 | ✅ 觸發 | policy 驗證；更新 pointsStatus |
