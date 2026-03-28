# 會員操作：合併與封存

## membermerge 會員合併檔

**TABLE_NAME**: `MEMBER_MERGE`
**檔案路徑**: `tables/memberRights/membermerge.ts`
**腳本路徑**: `tables/memberRights/scripts/membermerge/`
**JobQueues**: 5 個 queue，各 1 並發

### 用途

合併兩個會員帳號，保留 `destMemberName`（目的地會員），封存 `sourceMemberName`（來源會員）。

### 主要欄位

| 欄位 | 說明 |
|-----|-----|
| `destMemberName` | 保留會員編號（合併後留下的帳號） |
| `sourceMemberName` | 封存會員編號（合併後被封存的帳號） |
| `taskStatus` | 任務狀態：完成 / 錯誤 / 執行中 |
| `taskError` | 任務錯誤訊息 |

### 核心函式

| 函式 | 說明 |
|-----|-----|
| 會員合併（onJobRetry: 10） | 執行合併邏輯 |
| 已封存會員點數封存 | 封存被封存會員的點數帳本 |

### 排程

| 排程 | 頻率 | 說明 |
|-----|-----|-----|
| 會員合併排程 | FIVE_MINS | 定期執行待合併任務 |

### 合併邏輯（由 `merge.ts` 執行）

```
建立 membermerge 紀錄（destMemberName + sourceMemberName）
    ↓
policy 觸發 → 建立待辦工作（TodoJob）
    ↓
五分鐘排程觸發「會員合併」function
    ↓
1. 將 sourceMember 的相關資料移轉到 destMember：
   - pospet.memberName 更新為 destMemberName
   - pointaccount 中 sourceMember 的點數 → 轉移到 destMember
   - petparkcoupon 中 sourceMember 的優惠券 → 更新 memberName
   （依實際腳本邏輯而定）
    ↓
2. 封存 sourceMember（posmember 狀態改為封存）
    ↓
3. 封存 sourceMember 的點數帳本（archivePoint function）
    ↓
4. 在 posmember.pointArchivedAt 填上封存時間
    ↓
5. taskStatus 改為「完成」
```

### 注意事項

1. **防循環**：跨表更新時（如更新 pospet.memberName）需注意使用 `userContent` 旗標防止觸發迴圈
2. **onJobRetry: 10**：合併可能因資料量大而需要多次重試
3. **點數封存順序**：先執行合併，再封存點數帳本

---

## memberautoarchive 會員自動封存檔

**TABLE_NAME**: `MEMBER_AUTO_ARCHIVE`
**檔案路徑**: `tables/memberRights/memberautoarchive.ts`
**腳本路徑**: `tables/memberRights/scripts/memberautoarchive/`

### 用途

新增一筆紀錄後，系統會直接封存指定的 `memberName` 會員。這是封存會員的快速通道，無需手動操作 posmember 表格。

### 主要欄位

| 欄位 | 說明 |
|-----|-----|
| `memberName` | 待封存會員編號 |

### 腳本掛勾

| 掛勾 | 說明 |
|-----|-----|
| POLICY | 新增後直接封存（`scripts/memberautoarchive/policy.ts`） |

### 封存邏輯

```
建立 memberautoarchive 紀錄（memberName）
    ↓
POLICY 觸發
    ↓
直接呼叫封存 posmember API
    ↓
posmember 狀態改為封存（archived）
```

### 使用場景

- **會員合併後**：來源會員封存
- **異常帳號處理**：直接封存問題帳號
- **系統整合**：外部系統觸發封存操作
