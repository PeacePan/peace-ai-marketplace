# 硬體裝置與第三方整合

## `src/devices/` — 硬體周邊驅動程式

### `src/devices/creditCard/`
信用卡終端機整合，支援兩種收單機構：
- `skb.ts` — SKB（永豐/新光銀行）終端機通訊協定
- `tsb.ts` — TSB（台北富邦/台新）終端機通訊協定
- `index.ts` — 統一驅動介面
- `interface.ts` — 共用型別定義

### `src/devices/invoice/`
電子發票裝置整合。
- `index.ts` — 裝置通訊層
- `interface.ts` — 電子發票資料型別

### `src/devices/offline/`
- `syncEGUINo.ts` — 網路恢復時，同步本地端與遠端伺服器之間的電子發票流水號

### `src/devices/` 根目錄
- `mock.ts` — 匯出 `cMockPingResult`，用於裝置 ping 測試與模擬

## `src/thirdParty/` — 第三方服務整合

### `src/thirdParty/magento/`
Magento REST API 客戶端，用於線上訂單整合（MA Shop / 萬達美妝）。

| 檔案 | 用途 |
|------|------|
| `index.ts` | Magento API 主要客戶端 |
| `const.ts` | API 端點常數、狀態碼 |
| `_types.ts` | Magento 回應的 TypeScript 型別 |
| `mock/` | 開發與測試用的模擬資料 |

透過 `hooks/useMagentoAPI.ts` 使用，處理：
- MA Shop 訂單匯入（`maShopOrder`、`maPartialOrder`、`maFastBuyOrder`）
- 線上訂單狀態更新
- 線上訂單退換貨

### `src/thirdParty/taishinPay/`
Taishin One Pay 信用卡收單整合。

| 檔案 | 用途 |
|------|------|
| `index.ts` | Taishin Pay 客戶端 |
| `const.ts` | 顯示名稱、狀態碼 |
| `_types.ts` | 付款回應型別 |
| `utils.ts` | 輔助工具函式 |

透過 `hooks/useTaishinOnePay.ts` 使用。

### `src/thirdParty/edenred/`
Edenred 美食票支付整合。
- `index.tsx` — Edenred 付款流程元件
- `interface.ts` — Edenred API 型別
- `const.ts` — API 常數

使用於 `src/components/layouts/saleConsole/summary/edenredPay/`。

## `pages/api/` — 伺服器端 API 路由

| 路由檔案 | 端點 | 用途 |
|---------|------|------|
| `callExternalApi.pub.ts` | `POST /api/callExternalApi` | 代理有 CORS 限制的外部 API 呼叫 |
| `version.pub.ts` | `GET /api/version` | 取得目前應用程式版本 |
| `foodpanda/vendorOrders.pub.ts` | `POST /api/foodpanda/vendorOrders` | 接收 FoodPanda 外送訂單 Webhook |
| `pdf/itemReceipt.pub.ts` | `POST /api/pdf/itemReceipt` | 伺服器端產生商品收據 PDF |
| `pdf/shelf.pub.ts` | `POST /api/pdf/shelf` | 伺服器端產生貨架標籤 PDF |

> 另請參閱：[pages-and-routing.md](./pages-and-routing.md)，了解路由慣例與 CORS 設定。

`pages/api/` 中的輔助檔案：
- `_types.d.ts` — API 路由共用的請求/回應型別（如 `NextApiRequestParams`、`ShelfPDFAPIRequestBody`）
- `consts.ts` — `cAllowedOrigins` CORS 白名單
- `foodpanda/_types.d.ts` — 所有 FoodPanda API 型別定義

## FoodPanda 整合

FoodPanda 訂單流程：
1. `pages/api/foodpanda/vendorOrders.pub.ts` — 接收 Webhook
2. `src/contexts/foodpanda.ts` — 儲存並管理外送訂單
3. `src/components/tasks/pos/importDeliveryOrder/` — 匯入 POS 購物車的操作介面

相關 Model：`src/models/foodpandaStore.ts`

## 外送 / MA Shop 整合

Magento（MA）訂單：
- Models：`src/models/maShopOrder.ts`、`maPartialOrder.ts`、`maFastBuyOrder.ts`、`maShopPerformance.ts`、`maShopRefund.ts`、`maPartialExchange.ts`
- Context：`src/contexts/maShopOrder.ts`、`maPartialOrder.ts`
- Tasks：`src/components/tasks/pos/partialSearchOrExchange/`、`customerOrderSearchOrWithdraw/`

## `src/utils/mall-barcode/`
POS 商場條碼／QR Code 產生工具，支援跨店集點方案，相容多種商場格式：
- `index.ts` — 主要入口；處理標準商場條碼格式（如夢時代），回傳 `BARCODE` 或 `QRCODE` 結果型別
- `honghui.ts` — 宏匯商場 QR Code 格式（單一合併 QR，與標準格式不同）

規格文件位於 `docs/posmall-barcode/`。若要新增商場格式，請新增對應檔案並擴充 `index.ts`。

## `src/utils/pdf/`
PDF 產生工具，供 API 路由與用戶端列印共用：

| 檔案 | 用途 |
|------|------|
| `createItemReceiptPDF.ts` | 商品收據文件排版 |
| `createShelvesPDF.ts` | 貨架標籤文件排版 |
| `createVendorPurchaseOrderPDF.ts` | 廠商採購單文件 |
| `loadPDFmake.ts` | 延遲載入 pdfmake 以減少初始打包大小 |
| `mergePDFs.ts` | 使用 pdf-lib 合併多個 PDF 緩衝區 |

## `src/utils/tracking/`
數據分析與錯誤監控：
- `gtm.ts` — Google Tag Manager 事件輔助函式（`sendGTMEvent`）
- `sentry.ts` — Sentry 錯誤捕捉輔助函式（breadcrumb、capture）
- `interface.ts` — 追蹤事件型別定義

> 注意：Sentry 初始化位於 `src/utils/sentry/`（與追蹤輔助函式分開）。
