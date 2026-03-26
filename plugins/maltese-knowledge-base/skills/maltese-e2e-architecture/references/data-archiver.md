# Data Archiver 資料封存機制

## 概覽

E2E 測試會透過 GraphQL API 新增測試資料，測試結束後必須清理。Maltese 使用一個**自建 HTTP server**（port 50002）來協調 4 個平行 browser worker 的資料清理。

```
┌─────────────────────────┐
│   globalSetup           │
│   (port 50002 HTTP)     │
│                         │
│   toBeArchivedRecords   │
│   ┌──────────────────┐  │
│   │ tableName:       │  │
│   │   worker1: [...]  │  │
│   │   worker2: [...]  │  │
│   │   worker3: [...]  │  │
│   │   worker4: [...]  │  │
│   └──────────────────┘  │
└───────┬─────────────────┘
        │ HTTP API
   ┌────┼────┬────────┬─────────┐
   │    │    │        │         │
Worker1 Worker2  Worker3  Worker4
(posMain)(posETC) (salon) (workbench)
```

---

## HTTP Server（globalSetup.ts）

在 `globalSetup.ts` 的 `setup()` function 中啟動，監聽 `cTestPort + 1`（= 50002）。

### 端點

| Method | Path | 說明 |
|--------|------|------|
| POST | `/add` | 註冊待封存的資料紀錄 |
| GET | `/clean?jestWorkerId=N` | 封存指定 worker 的所有資料 |
| GET | `/clean` | 封存全部 worker 的所有資料 |
| GET | `/close` | 關閉 HTTP server |
| GET | `/failed` | 取得是否已標記測試失敗 |
| POST | `/failed` | 標記測試失敗 |

### POST /add

Request body:
```json
{
  "tableName": "item",
  "jestWorkerId": "1",
  "locators": [
    { "keyName": "_id", "keyValue": "abc123" }
  ]
}
```

將 locators 存入記憶體中的 `toBeArchivedRecords[tableName][jestWorkerId]`。

### GET /clean

1. 呼叫 `setupAdminJWT()` 取得管理員權限
2. 遍歷 `toBeArchivedRecords` 所有表格
3. 對每個表格的指定 worker（或全部 worker）收集 locators
4. 呼叫 `archiveUniAPI(tableName, locators)` 執行 GraphQL archive
5. **Double check：** 全部清除模式下，如果清除過程中又有新資料被加入，遞迴再次清除

### GET /close

關閉 HTTP server 並結束 process。

### GET/POST /failed

用於 `screenshotReporter.ts` 標記測試是否已失敗。避免多個 worker 同時截圖和清理。

---

## dataArchiver Client API

位於 `globalSetup.ts` 中匯出，供 `utils.ts` 和 `screenshotReporter.ts` 使用：

```typescript
export const dataArchiver = {
  /** 註冊待封存的紀錄 */
  addLocators: async (args: AddLocatorsArgs) => {
    await fetch(`${host}/add`, { method: 'POST', body: JSON.stringify(args) });
  },
  /** 清除指定 worker 的資料 */
  clean: async (jestWorkerId: string) => {
    await fetch(`${host}/clean?jestWorkerId=${jestWorkerId}`);
  },
  /** 清除所有資料 */
  cleanAll: async () => {
    await fetch(`${host}/clean`);
  },
  /** 關閉 server */
  close: async () => {
    await fetch(`${host}/close`);
  },
  /** 取得是否已標記失敗 */
  getIsTestFailed: async (): Promise<boolean> => { ... },
  /** 標記測試失敗 */
  setIsTestFailed: async () => { ... },
};
```

---

## 資料生命週期

```
insertUniAPI()
  ├── GraphQL insertV2 mutation → 資料寫入 JavaCat DB
  ├── if enableApproval → 遞迴簽核直到通過
  └── if archiveAfterTest (預設 true)
      └── dataArchiver.addLocators({ tableName, jestWorkerId, locators })
          └── POST /add → toBeArchivedRecords[table][worker].push(locators)

測試結束
  ├── 正常結束 → setupBrowserPage.afterAll()
  │   └── dataArchiver.clean(JEST_WORKER_ID)
  │       └── GET /clean?jestWorkerId=N
  │           └── archiveUniAPI() → GraphQL archiveV2 mutation → 軟刪除
  │
  ├── 全部正常結束 → globalTeardown
  │   ├── dataArchiver.cleanAll()  → 清除殘餘
  │   └── dataArchiver.close()     → 關閉 server
  │
  └── 測試失敗 → screenshotReporter
      ├── dataArchiver.setIsTestFailed()
      ├── takeScreenshot()
      ├── dataArchiver.cleanAll()  → 清除所有資料
      └── dataArchiver.close()     → 關閉 server
```

### archiveAfterTest: false

`000_dataInit.test.ts` 中的永久資料使用 `archiveAfterTest: false`，這些資料不會被註冊到 archiver，因此不會被清理：

```typescript
generateBatchInsertTestTask([{
  tableName: TABLE_NAME.VENDOR,
  displayName: TABLE_DISPLAY_NAME.VENDOR,
  records: [cMockItemVendor],
  options: { archiveAfterTest: false },  // 不清理
}]);
```

---

## screenshotReporter.ts

使用 Jasmine reporter API（非 Jest reporter）來監聽測試失敗：

```typescript
jasmine.getEnv().addReporter({
  specDone: (result) => {
    if (result.status === 'failed') {
      // 1. 印出失敗的 expectations
      console.error(result.failedExpectations);
      // 2. 串接 screenshot promise chain
      screenshotPromise = (async () => {
        await screenshotPromise;  // 等待前一個截圖完成
        if (!(await dataArchiver.getIsTestFailed())) {
          // 3. 標記失敗（避免其他 worker 重複處理）
          await dataArchiver.setIsTestFailed();
          // 4. 截圖
          await takeScreenshot(fullName, page);
          // 5. 封存所有資料
          await dataArchiver.cleanAll();
          // 6. 關閉 server
          await dataArchiver.close();
        }
      })();
    }
  },
});
```

**關鍵設計：**
- `beforeEach(() => screenshotPromise)` 確保截圖完成後才繼續
- `afterAll(() => screenshotPromise)` 確保測試結束前截圖完成
- 使用 `getIsTestFailed()` 防止多個 worker 同時執行失敗處理
- 失敗時會一併封存**所有** worker 的資料，而非只清理自己的

### takeScreenshot

```typescript
async function takeScreenshot(testName: string, pageInstance: Page) {
  if (pageInstance.isClosed()) return;
  // 截圖存到 tests/screenshots/ 目錄
  const buffer = await pageInstance.screenshot({ path: filePath, encoding: 'binary' });
  // CI 環境（CodeBuild）額外輸出 base64 DataURL 到 console
  if (process.env.CODEBUILD_PROJECT === 'Maltese') {
    console.log(`data:image/png;base64,${Buffer.from(buffer).toString('base64')}`);
  }
}
```

---

## 壓力測試（tests/e2e/stress/）

### 架構

```
stress/
├── index.test.ts      # 壓力測試進入點
├── checkout.test.ts   # 結帳壓力測試邏輯
├── browserLogs/       # 瀏覽器執行日誌
└── data/              # 壓力測試資料
    ├── items.json
    ├── memberNames.json
    ├── posconfig.json
    └── posstore.json
```

### 執行方式

```bash
BROWSERS=3 TASK_TIMES=5 npm run test:stress
```

- `BROWSERS` — 平行瀏覽器數量（預設 1）
- `TASK_TIMES` — 每個瀏覽器執行的任務次數（預設 1）
- timeout: 2 小時

### 流程

1. **主瀏覽器登入** — 使用 DEV 環境資料（非 mock）
2. **複製 N 個瀏覽器** — `cloneBrowser()` 複製 cookie + localStorage
3. **平行執行** — 每個瀏覽器執行相同結帳任務 M 次
4. **結果匯報** — 統計每個瀏覽器的 pass/fail 次數與失敗原因

### cloneBrowser 機制

```typescript
const destBrowser = await puppeteer.launch(puppeteerConfig.launch);
const destPage = (await destBrowser.pages())[0];
await destPage.emulateTimezone('Asia/Taipei');

// 複製 JavaCat GraphQL 端點的 cookie
const javaCatCookies = await sourcePage.cookies(`https://${javaCatDomain}/graphql`);
await destPage.setCookie(...javaCatCookies);

// 複製來源頁面的 cookie + localStorage
const sourceCookies = await sourcePage.cookies(sourceUrl);
const sourceLocalStorage = await sourcePage.evaluate(() => Object.assign({}, localStorage));
await destPage.goto(sourceUrl);
await destPage.setCookie(...sourceCookies);
await destPage.evaluate((storage) => {
  Object.entries(storage).forEach(([k, v]) => localStorage.setItem(k, v));
}, { ...sourceLocalStorage, ...overwriteLocalStorage });
```

**好處：** 複製後的瀏覽器已持有 JavaCat JWT token，不需要重新登入。可透過 `overwriteLocalStorage` 覆寫機台編號來模擬不同 POS 機台。

---

## GraphQL Client 設定（consts.ts）

```typescript
export const cSignRef = { jwt: '', cookie: '' };

export const cTestGraphQLClient = new ApolloClient({
  cache: new InMemoryCache({ addTypename: false }),
  link: new ApolloLink((operation, forward) => {
    // 攔截 response header 中的 set-cookie
    return asyncMap(forward(operation), async (response) => {
      const cookieToSet = headers.get('set-cookie');
      if (cookieToSet) {
        cSignRef.cookie = parsedCookies.map(({ name, value }) => `${name}=${value};`).join(' ');
      }
      return response;
    });
  }).concat(
    setLinkContext((_, { headers = {} }) => {
      // 每次請求帶上 jwt 和 cookie
      if (cSignRef.jwt) newHeaders.authorization = `Bearer ${cSignRef.jwt}`;
      if (cSignRef.cookie) newHeaders.cookie = cSignRef.cookie;
      return { mode: 'cors', headers: newHeaders };
    }).concat(
      createHttpLink({ uri: cGQLEndpoints.DEV, fetch, credentials: 'include' })
    )
  ),
  ssrMode: true,
});
```

**重點：**
- 使用 `cSignRef` 做為 JWT 和 Cookie 的跨請求共享容器
- 每次 GraphQL response 會自動更新 cookie
- 目標端點固定為 `cGQLEndpoints.DEV`（開發環境 JavaCat）
