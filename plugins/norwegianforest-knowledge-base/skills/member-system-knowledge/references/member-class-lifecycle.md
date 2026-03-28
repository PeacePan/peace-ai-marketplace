# 會員等級管理

## memberclass 會員等級設定檔

**TABLE_NAME**: `MEMBER_CLASS`
**檔案路徑**: `tables/memberRights/memberclass.ts`
**腳本路徑**: `tables/memberRights/scripts/memberclass/`

### 用途

設定升級金卡/黑卡所需的消費金額，以及續等的條件。每次修改等級規則，建立一筆新的 memberclass 紀錄，系統依時間（startAt～endAt）找到當前適用的設定。

### 主要欄位

| 欄位 | 說明 |
|-----|-----|
| `startAt` | 設定起始時間 |
| `endAt` | 設定結束時間 |
| `goldLevelUpSaleTotal` | 升級金卡所需消費金額 |
| `goldLevelUpMinSaleTotal` | 升級金卡最低消費金額（需搭配次數） |
| `goldLevelUpMinSaleCount` | 升級金卡最低消費次數（需搭配金額） |
| `goldHoldSaleTotal` | 續等金卡所需消費金額 |
| `goldHoldMinSaleTotal` | 續等金卡最低消費金額 |
| `goldHoldMinSaleCount` | 續等金卡最低消費次數 |
| `blackLevelUpSaleTotal` | 升級黑卡所需消費金額 |
| `blackLevelUpMinSaleTotal` | 升級黑卡最低消費金額 |
| `blackLevelUpMinSaleCount` | 升級黑卡最低消費次數 |
| `blackHoldSaleTotal` | 續等黑卡所需消費金額 |
| `blackHoldMinSaleTotal` | 續等黑卡最低消費金額 |
| `blackHoldMinSaleCount` | 續等黑卡最低消費次數 |

### 會員等級計算規則

```
會員等級判斷（在 posmember.updateMemberClass 中執行）：

1. 取得當前 memberclass 設定（startAt <= now <= endAt）
2. 讀取 posmember.consumerlifeTotal（會籍期間消費總金額）
3. 讀取 posmember.consumerlifeCount（會籍期間消費次數）

升等條件（金卡）：
  - consumerlifeTotal >= goldLevelUpSaleTotal
  - 若 goldLevelUpMinSaleTotal 有值：需同時 consumerlifeTotal >= goldLevelUpMinSaleTotal
    且 consumerlifeCount >= goldLevelUpMinSaleCount

升等條件（黑卡）：
  - consumerlifeTotal >= blackLevelUpSaleTotal
  - 若 blackLevelUpMinSaleTotal 有值：需同時 consumerlifeTotal >= blackLevelUpMinSaleTotal
    且 consumerlifeCount >= blackLevelUpMinSaleCount

續等條件（類似升等，但使用 Hold 欄位）
```

### 腳本掛勾

| 掛勾 | 檔案 | 說明 |
|-----|-----|-----|
| POLICY | `scripts/memberclass/policy.ts` | 驗證等級設定 |

---

## autoupdatememberclass 自動更新會員等級排程檔

**TABLE_NAME**: `AUTO_UPDATE_MEMBER_CLASS`
**檔案路徑**: `tables/memberRights/autoupdatememberclass.ts`
**腳本路徑**: `tables/memberRights/scripts/autoupdatememberclass/`
**JobQueues**: 3 個 queue，各 1 並發

### 用途

用於補跑歷史等級計算。當系統需要對特定日期的會員消費重新計算等級時，建立一筆 autoupdatememberclass 記錄，指定日期後啟動排程。

**常見使用場景**：
- 等級計算邏輯修正後，需重算歷史資料
- 系統上線初期的等級資料補算

### 主要欄位

| 欄位 | 說明 |
|-----|-----|
| `updateClassAt` | 更新會員等級日期（計算此日期的前一天消費） |
| `status` | 狀態：未執行 / 成功 / 失敗 / 取消 / 執行中 |

### 核心函式

| 函式 | 說明 |
|-----|-----|
| 更新日期 | 取得指定日期的會員消費，呼叫等級更新（`onJobRetry: 3`） |
| 開始任務 | 將狀態設為執行中，啟動排程 |
| 取消任務 | 將狀態設為取消，停止排程 |

### 排程

| 排程 | 頻率 | 說明 |
|-----|-----|-----|
| 更新日期排程 | TEN_MINS | 定期檢查並執行補跑任務 |

### 等級更新流程

```
autoupdatememberclass（建立紀錄）
    ↓ 開始任務（isPublic）
    ↓ 狀態設為「執行中」
    ↓ 排程每 10 分鐘觸發「更新日期」function
    ↓ 讀取指定日 00:00:00 ~ 23:59:59 的消費紀錄
    ↓ 批次呼叫 posmember.補更新會員等級
    ↓ 完成後狀態改為「成功」
```

---

## 會員等級整體流程

```
消費發生
    ↓
possale/posservicesale 完成結帳
    ↓
posmember.issuePoints（贈點）
    ↓ 同時
posmember.updateMemberClass 排程（FIVE_MINS）
    ↓
讀取 memberclass 當前有效設定
    ↓
計算 consumerlifeTotal vs 升等/續等門檻
    ↓
更新 posmember.memberClass / memberStartAt / memberEndAt
    ↓
若等級異動 → membershipLog push 紀錄
    ↓
policy 觸發 → issueClassChangeCoupon（發放等級異動優惠券）
    ↓
policy 觸發 → issueClassChangePoints（發放等級異動點數）
```
