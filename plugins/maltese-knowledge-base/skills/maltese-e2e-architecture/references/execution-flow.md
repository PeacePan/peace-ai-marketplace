# 測試執行流程

## NPM Scripts

```
npm run test:e2e        # CI 環境執行 E2E（headless）
npm run test:e2e:local  # 本地環境執行 E2E（headed，顯示瀏覽器）
npm run test:stress     # 壓力測試
```

## 執行步驟（bin/test.sh）

1. **設定環境變數**
   - `NODE_OPTIONS=--max-old-space-size=4096`
   - `NEXT_ENV=test`
   - `MALTESE_TARGET_ENV=dev`（CI）或 `local`（本地）
   - `PORT=50001`
   - `NODE_ENV=production`
   - `TZ=Asia/Taipei`

2. **前置檢查** — 確認 `.next` production build 存在，不存在則重新建置

3. **安裝 Chromium** — `npx puppeteer install chrome`

4. **資料初始化** — `jest --config tests/e2e/jest-e2e-data-init.config.ts`
   - 執行 `entries/000_dataInit.test.ts`
   - 建立永久測試資料（vendor、file category、user、role、item category、brand、animal、store、config 等）
   - 這些資料的 `archiveAfterTest: false`，測試結束後不會被清理

5. **執行 E2E 主測試** — `jest --config tests/e2e/jest-e2e.config.ts`
   - 啟動 4 個平行 Chrome browser
   - 各 browser 獨立執行對應的 entry file

6. **資料清理** — 殺掉殘餘 puppeteer / node server processes

---

## Jest Config 結構（jest-e2e.config.ts）

```
jest-e2e.config.ts
├── commonConfig
│   ├── testTimeout: 60000
│   ├── bail: true
│   ├── forceExit: true
│   └── verbose: true
├── projectConfig（每個 project 共用）
│   ├── testRunner: jest-jasmine2（非 jest-circus，為了 puppeteer 相容）
│   ├── setupFilesAfterEnv: [expect-puppeteer, screenshotReporter]
│   ├── roots: [tests/e2e/entries]
│   ├── globalSetup: jest-environment-puppeteer/setup
│   ├── globalTeardown: jest-environment-puppeteer/teardown
│   └── testEnvironment: jest-environment-puppeteer
├── projects（4 個平行瀏覽器）
│   ├── [0] posMain  — testRegex: posMain.test.ts  — 一般商品測試（電子發票）
│   ├── [1] posETC   — testRegex: posETC.test.ts   — 一般商品測試（收據）
│   ├── [2] salon    — testRegex: salon.test.ts     — 美容服務測試（電子發票）
│   └── [3] workbench — testRegex: workbench.test.ts — 美容工作臺+美容服務（收據）
└── taskConfig
    ├── preset: jest-puppeteer
    ├── globalSetup: tests/e2e/globalSetup.ts（自訂：資料封存 server）
    ├── globalTeardown: tests/e2e/globalTeardown.ts
    ├── maxWorkers: 4（= projects.length）
    └── notify: true（僅 local 環境）
```

**關鍵觀念：** `jest-e2e.config.ts` 有兩層 globalSetup：
- **外層** `taskConfig.globalSetup`：啟動資料封存 HTTP server（port 50002）
- **內層** `projectConfig.globalSetup`：jest-puppeteer 的瀏覽器啟動

---

## Puppeteer Config（jest-puppeteer.config.js）

```javascript
{
  browserContext: 'default',
  browserPerWorker: true,  // 每個 worker 各自一個 Chrome
  launch: {
    product: 'chrome',
    channel: 'chrome',
    headless: process.env.MALTESE_TARGET_ENV === 'local' ? false : 'new',
    defaultViewport: { width: 1024, height: 818 },  // 768 + 50px chrome bar
    ignoreHTTPSErrors: true,
  },
  exitOnPageError: true,  // 未捕獲的 page error 會中斷測試
  server: {
    command: 'ts-node -r tsconfig-paths/register -P tsconfig.server.json src/server/index.ts',
    port: 50001,         // 監聽此 port 確認 server ready
    host: 'localhost',
    usedPortAction: 'kill',
    launchTimeout: 60000,
  },
}
```

**重點：**
- `browserPerWorker: true` 讓每個 Jest worker（= 每個 project）擁有獨立的 Chrome 實體
- `exitOnPageError: true` 任何未捕獲的 page JS error 會終止測試
- Linux CI 環境自動加上 `--no-sandbox`, `--disable-setuid-sandbox`

---

## 全域 page 變數

jest-puppeteer 自動注入全域 `page` 變數到每個 test file：

```typescript
// 不需要 import，直接使用全域 page
it('example', async () => {
  await page.goto('http://localhost:50001');
  const elem = await page.$('#__next');
  expect(!!elem).toBeTruthy();
});
```

每個 Jest worker 的 `page` 是獨立的 Puppeteer Page 實體，綁定到該 worker 專屬的 Chrome browser。

---

## 環境區分

| 環境變數 | CI 值 | Local 值 | 影響 |
|---|---|---|---|
| `MALTESE_TARGET_ENV` | `dev` | `local` | headless vs headed；screenshot 輸出 base64 |
| `NODE_ENV` | `production` | `production` | Next.js production build |
| `NEXT_ENV` | `test` | `test` | GraphQL endpoint 選擇 |
| `CODEBUILD_PROJECT` | `Maltese` | (undefined) | CI 截圖輸出 base64 DataURL |
| `PORT` | `50001` | `50001` | 測試 server port |
| `TZ` | `Asia/Taipei` | `Asia/Taipei` | 時區設定 |
