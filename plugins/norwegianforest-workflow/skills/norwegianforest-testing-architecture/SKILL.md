---
name: norwegianforest-testing-architecture
description: >
  NorwegianForest 專案測試架構完整指南。涵蓋 Mocha + Chai 測試框架、Node.js VM Sandbox
  隔離執行機制、generateExecuteFunc 工具函式、QueryProcessor mock 模式、
  QueryEventEmitter 事件追蹤、NYC 覆蓋率儀表化，以及 Policy / Hook / Function / Approval /
  Cron / TodoJob 各類腳本的測試範例與最佳實踐。
  當需要為 NorwegianForest 撰寫新測試、修改既有測試、或理解測試架構時，務必先讀此 skill。
  適用場景包括：撰寫 policy 測試、撰寫 function 測試、撰寫 approval 測試、
  撰寫 cron 測試、撰寫 TodoJob 測試、除錯測試失敗、理解 VM sandbox mock 機制。
---

# NorwegianForest 測試架構

NorwegianForest 的表格腳本（policy、beforeInsert、beforeUpdate、function、approval、cron 等）在正式環境中運行於 JavaCat 的 **VM Sandbox**（基於 vm2 的 NodeVM）內。測試必須透過相同的 Sandbox 機制執行，才能確保測試結果與正式環境一致。

## 參考文件目錄

| 章節 | 內容摘要 | 何時閱讀 |
|------|---------|---------|
| [vm-sandbox-concept.md](references/vm-sandbox-concept.md) | VM 沙盒隔離機制、為什麼不能直接 import 測試、執行架構圖 | 第一次了解測試架構 |
| [tech-stack-and-structure.md](references/tech-stack-and-structure.md) | 技術堆疊、目錄結構、命名慣例、測試執行流程 | 建立新測試檔案或了解專案組織 |
| [core-utilities.md](references/core-utilities.md) | generateExecuteFunc、executeFuncUntilDoneOrError、findApprovalRule、writeRecord、sortRecords 完整 API | 使用工具函式時查閱參數與用法 |
| [query-processor-mock.md](references/query-processor-mock.md) | MySandbox.queryProcessor 覆寫模式、各 method 回傳值規範、QueryEventEmitter 事件追蹤 | 撰寫或修改 mock 查詢邏輯 |
| [test-examples.md](references/test-examples.md) | Policy、Hook、Function、Approval、Cron、TodoJob、BatchPolicy 七種腳本的完整測試範例 | 撰寫特定類型的測試時參考 |
| [checklist-and-pitfalls.md](references/checklist-and-pitfalls.md) | 撰寫測試 8 步清單、常見陷阱與注意事項 | 撰寫測試前的檢查清單、除錯測試失敗 |

## 快速定位

- **第一次寫測試** → 先讀 [vm-sandbox-concept.md](references/vm-sandbox-concept.md)，再讀 [checklist-and-pitfalls.md](references/checklist-and-pitfalls.md)
- **建立新測試檔案** → [tech-stack-and-structure.md](references/tech-stack-and-structure.md) 了解目錄與命名
- **設定 generateExecuteFunc** → [core-utilities.md](references/core-utilities.md)
- **撰寫 mock 查詢** → [query-processor-mock.md](references/query-processor-mock.md)
- **參考特定腳本測試寫法** → [test-examples.md](references/test-examples.md)
- **測試失敗除錯** → [checklist-and-pitfalls.md](references/checklist-and-pitfalls.md)

## 核心概念速覽

### 測試執行架構

```
測試程式 ──呼叫──▶ MySandbox.execute ──VM 沙盒執行──▶ 腳本邏輯
                                                          │
                                                    ctx.query.*
                                                          │
                                                    ◀── MessagePort ──▶
                                                          │
                                              MySandbox.queryProcessor（被 mock）
```

### 測試三要素

1. **`generateExecuteFunc`** — 讀取編譯後腳本、儀表化覆蓋率、回傳執行函式
2. **`MySandbox.queryProcessor`** — 攔截所有 `ctx.query.*` 呼叫並回傳 mock 資料
3. **`cloneDeep(mockData)`** — 每個測試案例使用獨立的 mock 資料副本

### 執行命令

```bash
npm test                              # 全部測試
npm test -- tests/possale             # 特定資料夾
npm test -- tests/possale/policy.test.ts  # 特定檔案
```
