# 第三方同步任務

## cresclabsyncmember CrescLab 同步會員

**TABLE_NAME**: `CRESC_LAB_SYNC_MEMBER`
**檔案路徑**: `tables/memberRights/cresclabsyncmember.ts`
**腳本路徑**: `tables/memberRights/scripts/cresclabsyncmember/`
**JobQueues**: 1 個 queue

### 用途

透過漸強（CrescLab）API 同步會員的 LINE 訂閱狀態，將結果更新到 `posmember.emLineIdStatus`。

### 主要欄位

| 欄位 | 說明 |
|-----|-----|
| `date` | 篩選日期 |
| `status` | 處理狀態：新建立 / 已下載 / 已同步 |
| `nextToken` | 漸強查詢分頁 Token（時效 30 分鐘，系統產生） |

### 表身

| 表身 | 說明 |
|-----|-----|
| `members` | 會員 LINE 資料（lineUID, lineUpdatedAt, status, syncStatus） |
| `members.syncStatus` | 已同步 / 已略過（找到會員同步，找不到略過） |
| 最大長度 | 6000 筆（避免下載 timeout，漸強一頁約 100KB） |

### 核心函式

| 函式 | 說明 |
|-----|-----|
| 取得漸強會員資料（isPublic） | 依任務條件取得資料並寫入表身 |
| LINE 同步會員狀態（isPublic） | 將 LINE 狀態同步至 posmember |

### 排程（僅 Production 環境）

| 排程 | 頻率 | 說明 |
|-----|-----|-----|
| 建立漸強存取任務 | HOURLY | 建立取得 LINE 狀態的排程任務 |
| 取得漸強會員資料 | HOURLY | 依任務條件下載漸強資料 |
| LINE 同步會員狀態 | HOURLY | 更新 posmember.emLineIdStatus |

### 同步流程

```
HOURLY 排程 → 建立漸強存取任務
    ↓
建立新的 cresclabsyncmember 紀錄（status = 新建立）
    ↓
HOURLY 排程 → 取得漸強會員資料
    ↓
呼叫漸強 API，以 nextToken 分頁取得最多 6000 筆會員 LINE 資料
    ↓
寫入 cresclabsyncmember.members 表身（status = 已下載）
    ↓
HOURLY 排程 → LINE 同步會員狀態
    ↓
逐筆讀取 members，依 lineUID 找到對應 posmember
    ↓
更新 posmember.emLineIdStatus
    ↓
cresclabsyncmember.status = 已同步
```

---

## ocardsyncpointtask OCard 會員同步點數任務檔（已棄用）

**TABLE_NAME**: `OCARD_SYNC_POINT_TASK`
**檔案路徑**: `tables/memberRights/ocardsyncpointtask.ts`
**狀態**: **@deprecated** 點數移轉完畢，已移除排程腳本，保留表格作歷史紀錄

### 歷史用途

OCard 平台的點數移轉任務，將 OCard 會員點數移轉到 NorwegianForest 系統。

### 相關 PR 參考

`https://github.com/petpetgo/petpetgo/pull/13644`

---

## ocardsynctask OCard 會員同步任務檔（已棄用）

**TABLE_NAME**: `OCARD_SYNC_TASK`
**檔案路徑**: `tables/memberRights/ocardsynctask.ts`
**狀態**: **@deprecated** 會員資料移轉完畢，已移除排程程式碼，保留表格作歷史紀錄

### 歷史用途

OCard 平台的會員資料移轉任務，將 OCard 會員資料同步到 NorwegianForest 系統。

### 相關 PR 參考

`https://github.com/petpetgo/petpetgo/pull/17330`

---

## 注意事項

1. **ocardsyncpointtask 與 ocardsynctask 已棄用**：
   這兩張表格僅保留作歷史紀錄，不應再新增任何腳本到這兩張表格。

2. **cresclabsyncmember 僅 Production 執行排程**：
   測試環境可手動呼叫 `isPublic` 的函式進行驗證，但排程只在 `isProduction` 時啟用。

3. **漸強 API Token 時效**：
   `nextToken` 時效僅 30 分鐘，若任務建立後超過 30 分鐘才執行「取得漸強會員資料」，Token 已失效需重新建立任務。
