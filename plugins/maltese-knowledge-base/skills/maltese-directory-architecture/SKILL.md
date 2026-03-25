---
name: maltese-directory-architecture
description: 在 Maltese POS 專案中工作時使用，用於了解功能應實作於何處、尋找既有程式碼，或理解各資料夾的用途
---

# Maltese 目錄架構

## 概覽

Maltese 是一套以 Next.js 14（Pages Router）、React 18、TypeScript、Apollo/GraphQL 及 MUI v5 建構的網頁版 POS（銷售時點）系統，支援零售 POS、美容 POS、電子發票、外送整合及離線模式。

## 資料夾快速對照表

| 路徑 | 用途 |
|------|---------|
| `pages/` | Next.js 路由。正式環境中只有 `.pub.tsx` 檔案為公開路由。 |
| `pages/api/` | 伺服器端 API 路由（外部 API Proxy、版本端點、FoodPanda Webhook、PDF 產生） |
| `src/algorithm/` | 純促銷組合求解器，不含 React 與任何副作用 |
| `src/components/elements/` | 通用可複用 UI 原子元件（鍵盤、搜尋列、對話框等） |
| `src/components/mui/` | 依照專案慣例封裝的 MUI 元件 |
| `src/components/layouts/` | 整頁版面框架（bootUp、saleConsole、workbench、setting） |
| `src/components/tasks/pos/` | POS 功能對話框（30+ 個：優惠券、退款、換貨等） |
| `src/components/tasks/salonPos/` | 美容 POS 功能對話框 |
| `src/components/tasks/common/` | 共用對話框（銷售查詢、日報表、開收班結帳等） |
| `src/components/widgets/` | 領域複合元件（payForSale、itemList、subtotal、member） |
| `src/config/` | 集中式執行期、裝置、GraphQL、業務及列印設定 |
| `src/contexts/` | React Context = 跨元件狀態 + GQL 輸入輸出（posSale、posMember、item、uiCtrller 等） |
| `src/devices/creditCard/` | 信用卡終端機驅動程式（SKB、TSB） |
| `src/devices/invoice/` | 電子發票裝置整合 |
| `src/devices/offline/` | 離線 EGUI 編號同步 |
| `src/graphql/` | Apollo client、查詢／異動輔助函式、Schema 型別、請求簽章 |
| `src/hooks/` | 自訂 React Hooks（useScene、useMagentoAPI、useInvoiceDevice 等） |
| `src/libs/` | 框架工廠：myFuncCompFactory（StateDecorator 包裝器）、stashArrayFactory |
| `src/models/` | 對應後端資料庫資料表的 TypeScript 介面（header + lines 模式） |
| `src/server/` | 支援 SSL 的自訂 Next.js/Express 伺服器 |
| `src/styles/` | 全域 MUI 主題與 SCSS Mixin |
| `src/thirdParty/magento/` | Magento REST API 客戶端（MA 線上訂單） |
| `src/thirdParty/taishinPay/` | 台新 One Pay 信用卡處理器 |
| `src/thirdParty/edenred/` | Edenred 餐飲券支付整合 |
| `src/utils/general/` | 純粹輔助函式：格式化、日期／期間、正規化、列印、稅務、裝置 |
| `src/utils/contexts/` | Context 使用的業務邏輯輔助函式（結帳、促銷、退貨） |
| `src/utils/gql/` | GQL 篩選條件與 limit-set 建構器 |
| `src/utils/pdf/` | PDF 產生（pdfmake + pdf-lib） |
| `src/utils/tracking/` | GTM 事件追蹤與 Sentry 錯誤輔助 |
| `src/utils/mall-barcode/` | POS 商場條碼解析（購物中心跨店紅利點數） |
| `src/utils/sentry/` | Sentry 錯誤監控初始化 |
| `src/workers/sharedWorker/` | 跨分頁 SharedWorker，用於即時同步 |
| `src/workers/webWorker/` | Web Worker，用於大量促銷計算 |
| `tests/e2e/` | Puppeteer E2E 測試，子目錄：posMain、posETC、salon、workbench、common |
| `tests/unit/` | Jest 單元測試，目錄結構對應 src/ |
| `bin/` | 開發、建置、部署及測試 Shell 腳本 |
| `public/` | 靜態資源：圖片、通知音效、PWA Manifest |
| `docs/` | 專案文件（條碼規格、實作計畫） |
| `etc/` | Docker 設定、AWS CodeBuild 規格、離線 Excel 範本 |

## 常見任務的實作位置

**新增 POS 功能對話框：** `src/components/tasks/pos/`（新建資料夾 + index.tsx）

**新增美容院對話框：** `src/components/tasks/salonPos/`

**新增 Workbench／POS 共用對話框：** `src/components/tasks/common/`

**新增頁面／路由：** `pages/`，正式環境可見的頁面使用 `.pub.tsx` 副檔名

**新增 Context（跨元件狀態）：** `src/contexts/`，遵循 `src/libs/` 中的 `myFuncCompFactory` 模式

**新增後端資料模型：** `src/models/`（header body 介面 + lines 介面 + 完整記錄型別）

**新增 GQL 查詢或異動：** `src/graphql/query.ts` 或 `src/graphql/mutation.tsx`

**新增支付硬體驅動程式：** `src/devices/creditCard/`（終端機驅動）、`src/devices/invoice/`（電子發票）、`src/devices/offline/`（離線同步）

**新增第三方服務：** `src/thirdParty/`

**新增工具函式：** `src/utils/general/`（純函式）或 `src/utils/contexts/`（與 Context 相關）

**新增可複用 UI 原子元件：** `src/components/elements/`

**新增自訂 Hook：** `src/hooks/`

## 參考文件

- [references/pages-and-routing.md](./references/pages-and-routing.md) — Pages 目錄、路由慣例、pub/dev 副檔名
- [references/components.md](./references/components.md) — 元件層級：elements、mui、layouts、tasks、widgets
- [references/state-and-data.md](./references/state-and-data.md) — Context、模型、GraphQL、Hooks、Workers
- [references/devices-and-integrations.md](./references/devices-and-integrations.md) — 硬體裝置、第三方服務、API 路由
- [references/utilities-and-tooling.md](./references/utilities-and-tooling.md) — 工具函式、設定、樣式、函式庫、測試、bin 腳本
