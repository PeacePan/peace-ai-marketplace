---
name: maltese-e2e-architecture
description: >
  Maltese POS 專案 E2E 測試架構完整知識庫。涵蓋 Jest+Puppeteer 測試框架設定、4 個平行測試套件
  （posMain / posETC / salon / workbench）的結構與執行順序、globalSetup 資料封存 HTTP server、
  GraphQL 測試資料操作、共用 helpers（clickByText / selectMuiMenuItem / toast / scrollToBottom）、
  Task-based 測試組織模式、Mock 資料結構，以及壓力測試架構。

  以下情況必須參考此文件再動手：
  - 修改或新增 E2E 測試案例
  - 修改測試基礎設施（globalSetup / globalTeardown / screenshotReporter）
  - 修改測試用 GraphQL helpers（insertUniAPI / archiveUniAPI / findBodyValueByLocators）
  - 修改共用 DOM helpers（clickByText / selectMuiMenuItem / getToastMessage / scrollToBottom）
  - 理解 4 個平行 browser 如何執行與隔離
  - 理解 data archiver server 如何在 globalSetup 啟動、各 worker 如何註冊/清理資料
  - 理解 Task function 如何組織為 Entry file 的執行順序
  - 撰寫壓力測試或理解 cloneBrowser 機制
  - 將 E2E 測試遷移至其他框架（如 Playwright）
---

# Maltese E2E 測試架構

## 概覽

Maltese E2E 測試使用 **Jest + Puppeteer**（jest-puppeteer 預設環境），以 4 個平行 Chrome 瀏覽器同時執行 4 個測試套件。測試透過 GraphQL API 建立/清理資料，每個瀏覽器在測試結束時自動封存（archive）建立的資料。

| 面向 | 技術 |
|------|------|
| Test Runner | Jest (jasmine2 runner) |
| Browser | Puppeteer (Chrome) |
| 平行化 | 4 Jest workers, 每個 worker 1 個 Chrome browser |
| Fail-fast | jasmine-fail-fast |
| GraphQL Client | Apollo Client |
| 資料清理 | 自建 HTTP server (port 50002) |
| 截圖 | 自製 screenshotReporter（CI 輸出 base64） |

## 核心檔案快速對照

| 路徑 | 用途 |
|------|------|
| `tests/e2e/jest-e2e.config.ts` | 主 E2E 設定：4 個 Jest projects + globalSetup/Teardown |
| `tests/e2e/jest-puppeteer.config.js` | Puppeteer 瀏覽器設定 + 測試 server 啟動 |
| `tests/e2e/jest-e2e-data-init.config.ts` | 資料初始化設定（E2E 前置作業） |
| `tests/e2e/jest-e2e-stress.config.ts` | 壓力測試設定 |
| `tests/e2e/consts.ts` | 測試常數 + Apollo Client + JWT/Cookie 管理 |
| `tests/e2e/utils.ts` | 所有共用函式（DOM helpers + GraphQL helpers） |
| `tests/e2e/globalSetup.ts` | 資料封存 HTTP server 啟動 |
| `tests/e2e/globalTeardown.ts` | 資料封存收尾 |
| `tests/e2e/screenshotReporter.ts` | 測試失敗時截圖 + 標記失敗 + 觸發資料封存 |
| `tests/e2e/entries/` | 測試進入點（4 個 entry files） |
| `tests/e2e/tasks/` | 測試邏輯實作（模組化 task functions） |
| `tests/e2e/mock/` | 測試用假資料（25 個 TypeScript 檔案） |
| `tests/e2e/stress/` | 壓力測試套件 |

## 參考文件

- [references/execution-flow.md](./references/execution-flow.md) — 測試執行流程、Jest config 結構、4 個平行 project 的設定、server 啟動、環境變數
- [references/task-organization.md](./references/task-organization.md) — Task-based 測試組織模式、Entry → Task 呼叫順序、4 個套件的完整 task 清單
- [references/helpers-and-utils.md](./references/helpers-and-utils.md) — DOM helpers（clickByText / selectMuiMenuItem / toast / scroll）、GraphQL helpers（insert / archive / find / call）、cloneBrowser
- [references/data-archiver.md](./references/data-archiver.md) — globalSetup HTTP server 架構、dataArchiver client API、資料生命週期、screenshotReporter 失敗處理
