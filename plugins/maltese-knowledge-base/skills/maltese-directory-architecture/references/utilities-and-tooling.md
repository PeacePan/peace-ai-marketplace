# 工具程式、設定與工具鏈

## `src/config/` — 集中式設定管理

所有執行期設定集中於此，統一透過 `src/config/index.ts` 匯出。

| 設定檔 | 內容 |
|--------|------|
| `runtime.config.ts` | `NEXT_ENV`、`MALTESE_TARGET_ENV`、依環境切換的功能旗標 |
| `storage.config.ts` | LocalStorage 與 SessionStorage 的鍵名常數（`cLocalStorageKey`、`cSessionStorageKey`）|
| `device.config.ts` | 周邊裝置設定（印表機類型、發票裝置類型）|
| `graphql.config.ts` | 各環境的 GQL 端點 URL |
| `thirdparty.config.ts` | 第三方服務端點與憑證設定 |
| `business.config.ts` | 商業邏輯常數（稅率、四捨五入規則等）|
| `print.config.ts` | 印表機專屬設定 |

**使用規範：** 設定值一律從 `src/config` 匯入，不得直接引用個別設定檔。

## `src/utils/general/` — 純粹輔助函式

不依賴 React，可在任何地方使用，包含 Worker 執行緒與測試環境。

| 檔案 | 內容 |
|------|------|
| `format.ts` | 數字／貨幣／日期格式化輔助函式 |
| `period.ts` | 日期區間產生、工作日判斷、`generateDateRangeConditions` |
| `normalize.ts` | 資料正規化（電子發票號碼驗證 `isEGUINo` 等）|
| `print.tsx` | 用戶端列印輔助函式 |
| `tax.ts` | 稅額計算輔助函式 |
| `device.ts` | 裝置偵測輔助函式 |
| `misc.ts` | 通用工具函式：`retry`、`sendParallelRequest`、debounce 輔助函式 |
| `mui.ts` | MUI 相關工具函式 |
| `eGUINo.ts` | 電子發票流水號管理輔助函式 |
| `field.ts` | 欄位／表單驗證輔助函式 |
| `statistics.ts` | 統計計算輔助函式 |

## `src/utils/gql/` — GQL 查詢條件建構器

用於組合 GraphQL 查詢條件的輔助工具：

| 檔案 | 用途 |
|------|------|
| `index.ts` | `convertRowAsNormFindCondition` — 將列資料轉換為 GQL 篩選條件 |
| `filter.ts` | 篩選條件建構器 |
| `limitSet.ts` | 分頁與筆數限制輔助函式 |
| `interface.d.ts` | 篩選條件型別定義 |
| `uploadFileToGraphQL.ts` | 透過 GQL mutation 上傳檔案 |

## `src/libs/` — 框架工廠函式

| 檔案 | 用途 |
|------|------|
| `myFuncCompFactory.tsx` | StateDecorator 包裝器 — Maltese 主要的狀態管理模式（詳見 `state-and-data.md`）|
| `fcFactory.tsx` | 函式元件工廠（myFuncCompFactory 的輕量版本）|
| `stashArrayFactory.ts` | 暫存陣列狀態工廠（用於暫存銷售功能）|
| `classes.ts` | CSS class 名稱組合輔助函式 |

## `src/styles/` — 全域樣式

| 檔案 | 用途 |
|------|------|
| `theme.ts` | MUI 主題設定（色彩、字型、斷點）|
| `mixin/layout.ts` | 以 JS-in-CSS 形式實作的版面 mixin（等同於 SCSS mixin）|

## 測試

### `tests/unit/` — 單元測試（Jest + React Testing Library）

目錄結構對應 `src/`：

| 目錄 | 測試對象 |
|------|----------|
| `tests/unit/algorithm/` | `src/algorithm/` 促銷組合邏輯 |
| `tests/unit/components/element/` | `src/components/elements/` UI 原子元件 |
| `tests/unit/components/mui/` | `src/components/mui/` 包裝元件 |
| `tests/unit/hooks/` | `src/hooks/` 自訂 Hook |
| `tests/unit/utils/` | `src/utils/` 純輔助函式 |
| `tests/unit/functions/` | 獨立函式測試 |

設定檔：`tests/unit/jest-unit.config.ts`

### `tests/e2e/` — 端對端測試（Puppeteer + jest-puppeteer）

```
tests/e2e/
├── entries/        # 測試進入點檔案（001_posMain.test.ts 等）
├── tasks/          # 功能測試實作
│   ├── posMain/    # POS 核心流程測試
│   ├── posETC/     # POS 雜項功能測試
│   ├── salon/      # 美容院 POS 功能測試
│   ├── workbench/  # 工作台操作測試
│   └── common/     # 共用測試工具／任務
├── mock/           # e2e 測試用模擬伺服器資料
├── stress/         # 負載／壓力測試腳本
└── utils.ts        # E2E 測試輔助工具
```

設定檔：`tests/e2e/jest-e2e.config.ts`、`tests/e2e/jest-e2e-data-init.config.ts`、`tests/e2e/jest-e2e-stress.config.ts`
Puppeteer 設定：`tests/e2e/jest-puppeteer.config.js`

### 執行測試

```bash
npm run test:unit          # 僅執行 Jest 單元測試
npm run test:e2e           # 執行 Puppeteer e2e 測試（需要伺服器運行中）
npm run test:e2e:local     # 對本地開發伺服器執行 e2e 測試
npm run test:stress        # 壓力測試
npm run tscheck            # TypeScript + ESLint 檢查
```

## `bin/` — Shell 腳本

| 腳本 | 用途 |
|------|------|
| `dev.sh` | 啟動開發伺服器（可透過 `--ssl` 啟用 SSL）|
| `build.sh` | 建置指定環境（`--local`、`--dev`、`--staging`、`--edge`，預設為正式環境）|
| `deploy.sh` | 部署至目標環境 |
| `start_by_branch.sh` | 依當前 git 分支啟動對應伺服器 |
| `test.sh` | 執行測試（接受 `--unit`、`--e2e`、`--local` 旗標）|
| `test-stress.sh` | 執行壓力測試套件 |
| `ssl.sh` | 產生本地 SSL 憑證 |

## `etc/` — 建置與部署設定

| 檔案／目錄 | 用途 |
|------------|------|
| `dockerfile` | 所有環境的 Docker 映像建置指令 |
| `buildspec.yml` | AWS CodeBuild 建置規格 |
| `offlineExcel/` | 離線模式資料匯出用 Excel 範本 |

## `docs/` — 專案文件

| 路徑 | 內容 |
|------|------|
| `docs/posmall-barcode/` | POS Mall 條碼格式規格（`README.md` 及條碼規格）|
| `docs/superpowers/plans/` | AI 輔助開發的實作計畫 |

## `public/` — 靜態資源

| 路徑 | 內容 |
|------|------|
| `public/images/` | 應用程式 Logo、外送圖示、載入動畫 |
| `public/sounds/` | 通知音效（`notification.mp3`、`miao-miao.mp3`、`fastBuyOrderNotify.mp3`）|
| `public/scripts/` | 下載的離線腳本（印表機驅動程式、離線 POS 銷售 zip）|
| `public/.well-known/` | PWA 與應用程式驗證檔案 |

## 外部共用函式庫（透過 webpack alias）

Maltese 透過 `next.config.js` 設定的 webpack alias，從相鄰的儲存庫匯入共用程式碼：

| Alias | 路徑 | 用途 |
|-------|------|------|
| `@norwegianForestLibs/*` | `../NorwegianForest/libs/` | 共用工具函式（欄位操作、資料正規化）|
| `@norwegianForestTables/*` | `../NorwegianForest/tables/` | 資料表名稱常數與列舉 |
| `@norwegianForestTypes/*` | `../NorwegianForest/libs/@types/` | 共用 TypeScript 型別（NormCondition、NormField 等）|
| `@xolo/*` | `../Xoloitzcuintli/` | 基礎介面型別 |
| `@oldEnglish/*`（POSAgent）| `../OldEnglish/POSAgent/src/` | POS agent 共用程式碼 |
| `@oldEnglishType` | `../OldEnglish/POSAgent/src/_type.ts` | POS agent 共用型別定義 |

這些 alias 同樣可透過 `tsconfig.json` 的 paths 設定使用。匯入時一律使用 alias 形式。

---
