# 頁面與路由

## Next.js Pages Router

Maltese 使用 **Pages Router**（而非 App Router）。所有路由以檔案形式定義於 `pages/` 目錄下。

## 副檔名規則

Next.js 在 `next.config.js` 中透過自訂 `pageExtensions` 進行設定：

| 副檔名 | 是否納入建置 | 用途 |
|--------|------------|------|
| `.pub.tsx` / `.pub.ts` | 永遠納入 | 公開正式環境路由 |
| `.dev.tsx` / `.dev.ts` | 僅限開發與本地環境 | 開發者沙箱路由 |
| 其他 `.tsx` / `.ts` | 永不作為路由 | 放置於 pages/ 的元件或輔助檔案 |

**規則：** `pages/` 內的檔案只有在副檔名為 `.pub.tsx` 或 `.pub.ts`（非正式環境則包含 `.dev.*`）時，才會成為路由。

## 路由結構

```
pages/
├── _app.pub.tsx            # 全域應用程式包裝器，所有 context provider 在此掛載
├── _document.pub.tsx       # 自訂 HTML 文件（Emotion SSR、MUI 設定）
├── index.pub.tsx           # 根路由 "/" → 主要 POS 銷售介面
├── checkout/
│   └── index.pub.tsx       # "/checkout" → 結帳流程
├── salon/
│   └── index.pub.tsx       # "/salon" → 美容 POS 介面
├── setting/
│   └── index.pub.tsx       # "/setting" → 店家設定
├── workbench/
│   └── index.pub.tsx       # "/workbench" → 工作台（日常營運）
├── offline/
│   └── app-auth-verify.pub.tsx  # "/offline/app-auth-verify" → 離線驗證檢查
└── api/
    ├── callExternalApi.pub.ts   # POST 代理，用於外部受 CORS 限制的 API
    ├── version.pub.ts           # 版本資訊端點
    ├── foodpanda/
    │   └── vendorOrders.pub.ts  # FoodPanda 訂單 webhook
    └── pdf/
        ├── itemReceipt.pub.ts   # 品項收據 PDF 產生
        └── shelf.pub.ts         # 貨架標籤 PDF 產生
```

## `__example/` 目錄

僅供開發人員使用的頁面，作為開發期間的沙箱環境，不會納入正式建置。適合用來獨立測試新元件。

## API 路由

所有 API 路由位於 `pages/api/` 下，同樣遵守 `.pub.ts` 副檔名規則。

主要行為：
- CORS 標頭在 `next.config.js` 中針對四個明確的來源進行設定：`https://data.wonderpet.asia`、`https://data-staging.wonderpet.asia`、`https://data-development.wonderpet.asia`、`https://data-local.wonderpet.asia`
- PDF 路由（`/api/pdf/*`）在伺服器端使用 `pdfmake` / `pdf-lib` 產生 PDF
- `callExternalApi.pub.ts` 作為後端代理，處理對外部服務的呼叫

## 自訂伺服器（`src/server/`）

`src/server/MyNextServer.ts` 以 Express 包裝 Next.js，僅用於本地開發環境（`npm run dev` / `npm run dev:ssl`）。它透過 `src/server/ssl/` 中的自簽憑證支援 HTTPS。正式環境直接使用 `next start -p 50000`，不啟用自訂伺服器。
