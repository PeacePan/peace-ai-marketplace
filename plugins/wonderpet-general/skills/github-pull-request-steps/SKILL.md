---
name: github-pull-request-steps
description: 通用 GitHub Pull Request 標準流程。涵蓋建立 Branch、開 PR、標上 Label、Code Review、E2E 測試驗收、更新 Label 到發送 Google Chat 摘要的完整步驟。適用於任何專案的 PR 管理流程。
---

# GitHub Pull Request 標準流程

本技能定義從第一個可提交單元完成後，一直到整個需求交付的 Pull Request 管理流程。

---

## 呼叫參數

使用此技能前，呼叫方需提供以下參數。標示「必填」的參數在流程中會被用到，標示「可選」的參數若未提供則使用預設行為。

| 參數 | 必填／可選 | 說明 | 預設值 |
|---|---|---|---|
| `branch-naming-rule` | 必填 | Git Branch 的命名規則，例如 `{prefix}-{ticket}-feature/{專案}/{描述}` | 無 |
| `pr-title-format` | 必填 | PR 標題的格式，例如 `[專案名稱][票號] {簡短描述}` | 無 |
| `ticket-url-prefix` | 必填 | Ticket 卡片連結的 URL 前綴，用於填入 PR 描述的卡片連結區塊 | 無 |
| `code-review-failure-assignee` | 必填 | Code Review 發現嚴重或高風險錯誤時，回報並重新發派的對象（開發者姓名或 subagent 名稱） | 無 |
| `e2e-qa-subagent` | 可選 | E2E 測試使用的 subagent 名稱。若未提供，Step 4 E2E 測試將跳過 | 無（跳過 E2E） |
| `e2e-qa-skill` | 可選 | E2E QA subagent 執行時需遵照的技能名稱 | 無 |
| `summary-title` | 可選 | Google Chat 摘要訊息的標題文字 | `【<專案名稱> 開發摘要】` |

---

## 環境前置檢查（每次開始工作前必做）

在進行任何 git / gh 操作前，必須先確認以下工具可用：

```bash
source ~/.bashrc
git --version
gh --version
```

- 若 `git` 不可用：請使用者安裝 [Git for Windows](https://git-scm.com/download/win) 並重新開啟終端機後再繼續。
- 若 `gh` 不可用：請使用者安裝 [GitHub CLI](https://cli.github.com/) 並執行 `gh auth login` 完成身分驗證後再繼續。

**工具未就緒時，不要嘗試自行修復 PATH 或繞過問題，直接告知使用者完成安裝與驗證後再回來。**

### husky pre-commit hook 失敗時的處理

若執行 `git commit` 時出現 `_/husky.sh: No such file or directory` 錯誤，執行：

```bash
cd .. && npx husky install
```

然後重新 commit。

---

## Step 1 — 第一個可提交單元完成時建立 Branch 與 Pull Request

若目前完成的是**第一個可提交單元**，在 commit 之前需先：

1. **確認 Ticket 編號**：若使用者未提供，必須主動詢問。
2. **建立 Git Branch**，依照 `branch-naming-rule` 參數定義的命名規則建立。
3. **建立 GitHub Pull Request**：
   ```bash
   git push -u origin <branch-name>
   gh pr create --title "<PR 標題>" --body "<PR 描述>"
   ```
   PR 標題依照 `pr-title-format` 參數格式填入。
   PR 描述請按照以下範本：

   ```markdown
   ## 摘要
   <!--
   簡述如何實作此功能，如果是錯誤修正，請說明錯誤的發生原因
   -->
   {描述在此處}

   ## 卡片連結
   {ticket-url-prefix}/{ticket}

   ## 提醒事項
   ### 📋 開發者提醒事項
   - 確認 PR 標題描述正確
   - 確認 Merge base
   - 確認 Label 標示正確
   ### 📋 審查者提醒事項
   - 確認 merge target
   - 確認 Label 更改完成
   - 確認代碼註解是否完善
   - 列出建議修正項目
   - 列出建議效能優化項目
   ```

4. 執行 `git push` 將 commit 推送至遠端。

---

## Step 2 — 為 PR 標上 `working` Label

PR 建立後，立即標上 `working` label：

```bash
gh pr edit <PR-number> --add-label "working"
```

---

## Step 3 — 所有可提交單元完成後進行 Code Review

當所有工作均完成後，使用 `wonderpet-general:code-review-principles` 對此 PR 進行審查：

- 若出現**嚴重（critical）或高風險（high）程度的錯誤**，必須回報給 `code-review-failure-assignee` 參數指定的對象重新實作。
- 重新實作後再次進行 Code Review，直到沒有嚴重或高風險錯誤為止。

---

## Step 4 — 若有 UI 改動且專案支援 E2E 測試，進行 E2E 測試（可選）

> 此步驟為**可選**。若呼叫方未提供 `e2e-qa-subagent` 參數，直接跳過此步驟。

若此次需求包含任何 UI 改動，且 `e2e-qa-subagent` 參數已提供：

- 將實作結果發派給 `e2e-qa-subagent` 參數指定的 subagent 進行 E2E 測試。
- 若有提供 `e2e-qa-skill` 參數，**發派時必須明確指示 agent 遵照該技能的完整流程**。
- 若測試**未通過**，回報給 `code-review-failure-assignee` 參數指定的對象進行調整，直到測試通過。
- 若測試**通過**，進入下一步。

---

## Step 5 — 更新 PR Label 為 `done`

整個需求開發完畢後：

```bash
gh pr edit <PR-number> --remove-label "working" --add-label "done"
```

---

## Step 6 — 發送摘要至 Google Chat

將整個實作結果的摘要，以**條列式、1000 字以內**的**繁體中文**訊息，發送至 Google Chat 聊天室。

> ⚠️ **不可使用 `curl` 傳送含中文的訊息**，Windows Git Bash 下 curl 傳遞中文字串會產生亂碼。**必須使用 Node.js** 發送：

```bash
node -e "
const https = require('https');
const url = new URL('https://chat.googleapis.com/v1/spaces/AAQABhe-wqI/messages?key=AIzaSyDdI0hCZtE6vySjMm-WEfRq3CPzqKqqsHI&token=NsFjTXa0wJ1flTCW2CTMdgtFZqWFqX4lHojhwDQwAp0');
const body = JSON.stringify({ text: '<summary-title 參數>\n\n<條列式摘要內容>' });
const req = https.request({ hostname: url.hostname, path: url.pathname + url.search, method: 'POST', headers: { 'Content-Type': 'application/json; charset=utf-8' } }, res => {
  let data = '';
  res.on('data', chunk => data += chunk);
  res.on('end', () => console.log('sent:', JSON.parse(data).name));
});
req.write(body);
req.end();
"
```

摘要內容應包含：
- 完成的功能清單
- 變更的主要檔案或模組
- 重要的設計決策
- 測試結果
