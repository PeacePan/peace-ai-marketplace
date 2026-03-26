# 技術堆疊與目錄結構

## 技術堆疊

| 元件 | 版本/工具 | 用途 |
|------|----------|------|
| 測試框架 | **Mocha 10** | 測試執行器，支援平行執行（`--parallel --jobs 2`） |
| 斷言庫 | **Chai 4** | `expect` 風格斷言 |
| 覆蓋率 | **NYC (Istanbul) 15** | 腳本儀表化 + 覆蓋率報告 |
| TypeScript | **ts-node** (transpile-only) | 測試檔案直接使用 TypeScript |
| 沙盒執行 | **vm2 NodeVM** (透過 JavaCat MySandbox) | 腳本隔離執行 |
| 資料複製 | **lodash.cloneDeep** | 每個測試案例的 mock 資料獨立 |

## 目錄結構

```
NorwegianForest/
├── bin/test.sh                    # 測試啟動腳本
├── .nycrc                         # NYC 覆蓋率設定
├── tests/
│   ├── global-setup.ts            # 全域變數初始化（dayjs, Utils, Procurement）
│   ├── utils.ts                   # 測試工具函式（核心）
│   ├── example.test.ts            # 範例測試（新手參考）
│   ├── env.testing.norcat.yml     # 測試環境變數
│   │
│   ├── possale/                   # ← 每個表格一個資料夾
│   │   ├── mocks.ts              # mock 資料定義
│   │   ├── policy.test.ts        # policy 腳本測試
│   │   ├── beforeInsert.test.ts  # beforeInsert 腳本測試
│   │   ├── beforeUpdate.test.ts  # beforeUpdate 腳本測試
│   │   └── functions/            # function 腳本測試
│   │       ├── balance.test.ts
│   │       └── finishCheckout.test.ts
│   │
│   ├── requisition/
│   │   ├── approval.test.ts      # 簽核鏈測試
│   │   └── functions/
│   │       └── req2po.test.ts
│   │
│   ├── batchmutation/
│   │   ├── mocks.ts
│   │   └── functions/
│   │       └── process.test.ts   # TodoJob 函式測試
│   │
│   └── vendor/
│       ├── note.cron.test.ts     # cron 腳本測試（直接 import 函式）
│       └── monthlyNote.cron.test.ts
│
├── tables/                        # 被測試的腳本原始碼
│   ├── pos/scripts/possale/
│   │   ├── policy.ts
│   │   ├── beforeInsert.ts
│   │   ├── beforeUpdate.ts
│   │   └── ...
│   └── common/                    # 共用函式庫（會以 libs 注入 VM）
│       ├── dayjs.ts
│       ├── utils.ts
│       └── validator.ts
```

## 命名慣例

| 類型 | 檔名格式 | 範例 |
|------|---------|------|
| Policy 測試 | `policy.test.ts` | `tests/possale/policy.test.ts` |
| BeforeInsert 測試 | `beforeInsert.test.ts` | `tests/possale/beforeInsert.test.ts` |
| BeforeUpdate 測試 | `beforeUpdate.test.ts` | `tests/possale/beforeUpdate.test.ts` |
| Function 測試 | `functions/<funcName>.test.ts` | `tests/possale/functions/balance.test.ts` |
| Approval 測試 | `approval.test.ts` | `tests/requisition/approval.test.ts` |
| BatchPolicy 測試 | `batchPolicy.test.ts` | `tests/item/batchPolicy.test.ts` |
| Cron 測試 | `<name>.cron.test.ts` | `tests/vendor/note.cron.test.ts` |
| Mock 資料 | `mocks.ts` | `tests/possale/mocks.ts` |

## 測試執行流程

### 執行命令

```bash
# 執行全部測試
npm test

# 執行特定資料夾
npm test -- tests/possale

# 執行特定檔案
npm test -- tests/possale/policy.test.ts
```

### 啟動流程（bin/test.sh）

```bash
# 1. 建置 JavaCat worker threads pool
cd ../JavaCat && npm run build:worker

# 2. 設定環境變數
export TZ=Greenwich           # 統一時區
export SANDBOX_QUIET=1        # 關閉 Sandbox 冗餘訊息

# 3. 建置 NorwegianForest（webpack 編譯腳本）
cd ../NorwegianForest && npm run build

# 4. 載入環境設定檔
export MOCHA_ENV_FILES="...env.testing.yml,...env.testing.norcat.yml"

# 5. 執行 Mocha（平行 2 個 worker、30 秒逾時）
npx nyc mocha --parallel --jobs 2 \
  -r ts-node/register/transpile-only \
  -r ../JavaCat/test/envYamlLoader.ts \
  -r ../JavaCat/test/mocha.fixture.ts \
  -r tsconfig-paths/register \
  -r tests/global-setup.ts \
  -s 5000 -t 30000 --exit \
  "${TARGET_DIR}"
```

### Mocha 載入順序

| 順序 | 模組 | 功能 |
|------|------|------|
| 1 | `ts-node/register/transpile-only` | TypeScript 編譯（僅轉譯，不做型別檢查） |
| 2 | `../JavaCat/test/envYamlLoader.ts` | 載入 YAML 環境設定到 `process.env` |
| 3 | `../JavaCat/test/mocha.fixture.ts` | 初始化 `global.testTools`（context、factory 等基礎設施） |
| 4 | `tsconfig-paths/register` | 路徑別名解析（`@javaCat/*`、`@norwegianForestTables/*` 等） |
| 5 | `tests/global-setup.ts` | 注入全域變數（`global.dayjs`、`global.Utils`、`global.Procurement`） |

### NYC 覆蓋率設定（.nycrc）

```json
{
  "extends": "@istanbuljs/nyc-config-typescript",
  "all": false,
  "check-coverage": false,
  "reporter": ["text", "text-summary"],
  "include": ["tables/**"],
  "exclude": ["**/_type.ts"]
}
```

- 覆蓋率範圍：`tables/` 底下的所有檔案
- 排除型別定義檔（`_type.ts`）
- 不強制覆蓋率門檻（`check-coverage: false`）

### global-setup.ts

```typescript
// 因為測試到共用函式庫，原本表格定義裡的 libs 的 namespace
// 會在 JavaCat 的 VM Sandbox 被統一注入在全域變數裡，
// 因此在測試共用函式庫前必須自行宣告全域變數。
global.dayjs = dayjs;
global.Procurement = Procurement;
global.Utils = Utils;
```

如果你的腳本依賴其他全域變數，需要在此檔案中補充。
