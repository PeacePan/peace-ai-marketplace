---
name: ragdoll-develop-workflow
description: 定義 Ragdoll 專案的開發工作流程，確保開發過程中的協作效率和代碼質量。此技能涵蓋從任務分配、代碼實作、測試到最終合併的整個流程，並提供相關工具和最佳實踐指導。
---

# Ragdoll AI Agent 開發工作流程

## 流程概覽（Mermaid 視覺化）

```mermaid
flowchart TD
    A([使用者提出規格需求]) --> B["using-superpowers<br>蒐集需求與工具"]
    B --> C[與使用者確認 Spec]
    C --> D{Spec 確認？}
    D -- 否 --> C
    D -- 是 --> E["擬定 Plan<br>切分成多個 Chunk"]
    E --> F{Plan 與 Spec 相符？}
    F -- 否 --> E
    F -- 是 --> G[["發派 Chunk 任務<br>ragdoll-electron-rd<br>ragdoll-next-rd"]]

    G --> QA[["交由 QA 驗證<br>ragdoll-electron-qa<br>ragdoll-next-qa"]]
    QA --> QAR{QA 測試通過？}
    QAR -- 否 --> QAFix[["回報 RD 修正<br>ragdoll-electron-rd<br>ragdoll-next-rd"]]
    QAFix --> QA
    QAR -- 是 --> H{是否為第一個 Chunk？}

    H -- 是 --> I{"使用者有提供<br>Jira Ticket？"}
    I -- 否 --> J["詢問使用者<br>Jira Ticket 編號"]
    J --> K
    I -- 是 --> K["開立 Git Branch<br>RD-{ticket}-feature/ragdoll/{描述}"]
    K --> L[建立 GitHub Pull Request]
    L --> M["PR 標上 label: working"]
    M --> N["git add + git commit<br>git push"]

    H -- 否 --> O["git add + git commit"]

    N --> P{還有下一個 Chunk？}
    O --> P
    P -- 是 --> G
    P -- 否 --> Q[所有 Chunk 完成]

    Q --> R["code-review<br>對 PR 進行 Code Review"]
    R --> S{有嚴重或高風險錯誤？}
    S -- 是 --> G
    S -- 否 --> T{需求包含 UI 改動？}

    T -- 否 --> U["移除 label: working<br>標上 label: done"]
    T -- 是 --> V[["Agent ragdoll-e2e-qa<br>進行 E2E 測試"]]
    V --> W{E2E 測試通過？}
    W -- 否 --> G
    W -- 是 --> U

    U --> X[發送摘要至 Google Chat]
    X --> Y([完成])
```

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

## 詳細流程說明

### Step 1 — 接收需求，呼叫 `/using-superpowers`

當使用者提出規格開發需求時，**必須先呼叫 SKILL `/using-superpowers`** 來蒐集相關工具與上下文，再進入後續流程。

---

### Step 2 — 與使用者確認 Spec

- 整理使用者需求，輸出清楚的功能規格（Spec）。
- 與使用者反覆確認，直到雙方對 Spec 內容達成共識。

---

### Step 3 — 擬定 Plan 並切分 Chunk

- 根據確認好的 Spec 擬定實作計畫（Plan）。
- Plan 必須切分為多個 **Chunk**（子任務），每個 Chunk 對應可獨立提交的工作單元。
- 確認 Plan 的每個 Chunk 均能對應到 Spec 的功能需求，確保沒有遺漏。

---

### Step 4 — 發派 Chunk 任務給 Subagent

依照 Chunk 的內容，將任務分派給對應的 subagent：

| Subagent | 負責範圍 |
|---|---|
| `ragdoll-electron-rd` | Electron 層：SQLite、IPC、背景排程、Node.js 後端邏輯 |
| `ragdoll-next-rd` | Next.js 層：前端 UI、資料層、Store 串接 |

> 若 Chunk 同時涉及兩層，兩個 subagent 可並行發派。

---

### Step 5 — 交由 QA Subagent 進行測試驗證

當 `ragdoll-electron-rd` 或 `ragdoll-next-rd` 完成 Chunk 實作後，**必須**將實作結果交由對應的 QA subagent 進行測試：

| RD Subagent | 對應 QA Subagent |
|---|---|
| `ragdoll-electron-rd` | `ragdoll-electron-qa` |
| `ragdoll-next-rd` | `ragdoll-next-qa` |

> 若 Chunk 同時涉及兩層，兩個 QA subagent 可並行發派。

**測試未通過時的處理流程：**

1. QA subagent 回報測試失敗的詳細錯誤訊息與失敗原因。
2. 將錯誤資訊轉交給對應的 RD subagent 進行修正。
3. RD subagent 修正完成後，再次交由 QA subagent 驗證。
4. **重複上述步驟，直到 QA 測試全部通過為止。**

測試通過後，才能進入下一步的 Git Commit。

---

### Step 6 — 每個 Chunk 完成後進行 Git Commit

每當 subagent 完成一個 Chunk 的實作，立即執行：

```bash
git add <相關檔案>
git commit -m "<清楚描述此 Chunk 的變更>"
```

---

### Step 7 — 第一個 Chunk 完成時建立 Branch 與 Pull Request

若目前完成的是**第一個 Chunk**，在 commit 之前需先：

1. **確認 Jira Ticket 編號**：若使用者未提供，必須主動詢問。
2. **建立 Git Branch**，命名規則：
   ```
   RD-{jira-ticket}-feature/ragdoll/{30字以內的描述}
   ```
   範例：`RD-1234-feature/ragdoll/checkout-discount-calculator`

3. **建立 GitHub Pull Request**：
  ```bash
  git push -u origin <branch-name>
  gh pr create --title "<PR 標題>" --body "<PR 描述>"
  ```
  PR 標題的格式為 `[Ragdoll][RD-6945] {簡短描述}`
  PR 描述請按照範本

  ```markdown
  ## 摘要
  <!--
  簡述如何實作此功能，如果是錯誤修正，請說明錯誤的發生原因
  -->
  {描述在此處}

  ## 卡片連結
  https://wonderpet.atlassian.net/browse/{jira-ticket}
  ## 提醒事項
  ### 📋 開發者提醒事項
  - 確認 PR 標題描述正確
  - 確認 Merge base
  - 確認 Label 標示正確
  - Hotfix 表格變更 Merge Sprint 表格分支後再 Patch 到 Dev 表格
  - Hotfix 表格變更向下 Merge 後要再 Patch 到各個分支（Staging、Dev）的表格
  ### 📋 審查者提醒事項
  - 確認 merge target
  - 確認 Label 更改完成
  - 確認代碼註解是否完善
  - 列出建議修正項目
  - 列出建議效能優化項目
  ```

4. 執行 `git push` 將 commit 推送至遠端。

---

### Step 8 — 為 PR 標上 `working` Label

PR 建立後，立即標上 `working` label：

```bash
gh pr edit <PR-number> --add-label "working"
```

---

### Step 9 — 所有 Chunk 完成後進行 Code Review

當所有 Chunk 均完成後，呼叫 `/code-review` 對此 PR 進行審查：

- 若出現**嚴重（critical）或高風險（high）程度的錯誤**，必須回報給 `ragdoll-electron-rd` 與 `ragdoll-next-rd` 重新實作。
- 重新實作後再次進行 Code Review，直到沒有嚴重或高風險錯誤為止。

---

### Step 10 — 若有 UI 改動，交由 E2E QA 測試

若此次需求包含任何 UI 改動：

- 將實作結果發派給 subagent `ragdoll-e2e-qa` 進行 E2E 測試。
- **發派時必須明確指示 agent 遵照 `ragdoll-e2e-workflow` 技能的完整流程**，包含先讀取 `ragdoll-checkout-flow` 與 `playwright-best-practices`，以及到 `test-results/` 查看截圖和錯誤訊息診斷失敗原因。
- 若測試**未通過**，重新發派任務給 `ragdoll-electron-rd` 與 `ragdoll-next-rd` 進行調整，直到測試通過。
- 若測試**通過**，進入下一步。

---

### Step 11 — 更新 PR Label 為 `done`

整個需求開發完畢後：

```bash
gh pr edit <PR-number> --remove-label "working" --add-label "done"
```

---

### Step 12 — 發送摘要至 Google Chat

將整個實作結果的摘要，以**條列式、1000 字以內**的訊息，發送至 Google Chat 聊天室。

> ⚠️ **不可使用 `curl` 傳送含中文的訊息**，Windows Git Bash 下 curl 傳遞中文字串會產生亂碼。**必須使用 Python** 發送：

```bash
python3 - << 'PYEOF'
import json, urllib.request

msg = {
    "text": "【Ragdoll 開發摘要】\n\n<條列式摘要內容>"
}
url = "https://chat.googleapis.com/v1/spaces/AAQABhe-wqI/messages?key=AIzaSyDdI0hCZtE6vySjMm-WEfRq3CPzqKqqsHI&token=NsFjTXa0wJ1flTCW2CTMdgtFZqWFqX4lHojhwDQwAp0"
data = json.dumps(msg, ensure_ascii=False).encode("utf-8")
req = urllib.request.Request(url, data=data, headers={"Content-Type": "application/json; charset=utf-8"})
with urllib.request.urlopen(req) as resp:
    result = json.loads(resp.read())
    print("sent:", result.get("name"))
PYEOF
```

> **Python 指令因環境而異：**
> - **macOS**：使用 `python3`（系統內建或透過 Homebrew 安裝）。
> - **Windows Git Bash**：`python3` 指向 Windows Store 版本（無法使用），請改用 `python`（位於 `/c/Python312/python`）。
>
> 執行前可先用 `python3 --version || python --version` 確認可用的指令。

摘要內容應包含：
- 完成的功能清單
- 變更的主要檔案或模組
- 重要的設計決策
- 測試結果

---

## Subagent 對照表

| Subagent | 角色 | 技術範疇 |
|---|---|---|
| `ragdoll-electron-rd` | Electron 開發 | SQLite、IPC、Node.js 後端 |
| `ragdoll-next-rd` | Next.js 開發 | 前端 UI、Store、API 串接 |
| `ragdoll-e2e-qa` | E2E 測試 | Playwright、結帳流程測試 |
| `ragdoll-electron-qa` | Electron 測試 | Electron 層功能驗證 |
| `ragdoll-next-qa` | Next.js 測試 | Next.js 層功能驗證 |

---

## 分支命名規則

```
RD-{jira-ticket}-feature/ragdoll/{30個字以內的描述}
```

- `{jira-ticket}`: Jira Ticket 編號（若使用者未提供，必須詢問）
- `{描述}`: 以連字號分隔的英文短描述，30 字以內

**範例：**
- `RD-6857-feature/ragdoll/checkout-e2e-testing`
- `RD-1234-feature/ragdoll/discount-calculator-fix`
