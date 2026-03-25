# 元件架構

## 層級結構

```
src/components/
├── elements/    # 第一層：通用 UI 原子元件 — 不含業務邏輯
├── mui/         # 第一層：MUI 封裝元件 — 在 MUI 之上套用專案慣例
├── widgets/     # 第二層：領域複合元件 — 具備業務邏輯，可跨版面重用
├── layouts/     # 第三層：完整頁面版面框架 — 組合 widgets 與 tasks
└── tasks/       # 第三層：功能性對話框/抽屜 — 由版面操作觸發開啟
```

## `elements/` — 通用 UI 原子元件

不含 POS 業務邏輯的可重用元件，可安全用於任何地方。

| 元件 | 用途 |
|-----------|---------|
| `BootUpLoading` | 應用程式初始化期間顯示的全螢幕載入畫面 |
| `BtnList` | 標準化按鈕清單容器 |
| `BtnToolbar` | 標準化工具列容器 |
| `DateSearch` | 具備 POS 風格介面的日期範圍選擇器 |
| `DebounceButton` | 防止重複點擊送出的按鈕 |
| `EnvMark` | 顯示目前環境（dev/staging）的視覺標記 |
| `ErrorBoundary` | React 錯誤邊界封裝器 |
| `NumberKB` | 標準螢幕數字鍵盤 |
| `ExNumberKB` | 擴充版螢幕數字鍵盤 |
| `TableKB` | 嵌入表格的數字鍵盤 |
| `NumberInput` | 受控數字輸入欄位 |
| `MultipleProviders` | 用於組合多個 React context provider 的工具元件 |
| `MultiSourceTable` | 合併多個資料來源的表格 |
| `ScrollContainer` | 具備自訂捲動行為的容器 |
| `SearchBar` | 附防抖功能的搜尋輸入框 |
| `SimpleConfirmDialog` | 可重用的是/否確認對話框 |
| `ThemeLoading` | 套用應用程式主題樣式的載入轉圈動畫 |

## `mui/` — MUI 封裝元件

針對 MUI 元件進行專案級封裝，統一 props 介面與樣式規範。

| 元件 | 用途 |
|-----------|---------|
| `MuiDialog` | 套用專案慣例的模態對話框 |
| `MuiForm` | 附驗證輔助功能的表單容器 |
| `MuiGridLayout` | 使用 MUI Grid 的響應式格線佈局 |
| `MuiTab` | 分頁導覽元件 |
| `MuiTable` | 支援排序與分頁的資料表格 |
| `MuiToast` | Snackbar/toast 通知系統 |

## `widgets/` — 領域複合元件

具備業務邏輯、可跨多個版面共用的元件。每個 widget 封裝一個領域概念。

| Widget | 領域 |
|--------|--------|
| `checkEGUIBarcode` | 電子發票條碼驗證 |
| `creditNote` | 折讓單顯示與輸入 |
| `itemList` | 商品清單面板（主要購物車視圖） |
| `member` | 會員資訊顯示面板 |
| `memberValidation` | 會員 ID/電話驗證流程 |
| `multiServiceItemTable` | 美容院服務項目表格 |
| `payForSale` | 付款方式選擇與處理 |
| `printPriceCard` | 價格標籤列印介面 |
| `promotionView` | 促銷活動詳情顯示 |
| `refund` | 退貨項目選擇與處理 |
| `returnTableView` | 退貨訂單表格 |
| `searchMemberView` | 會員搜尋介面 |
| `selectPromotionTab` | 促銷活動選擇分頁 |
| `subtotal` | 購物車小計/摘要面板 |
| `user` | 使用者（員工）登入面板 |

## `layouts/` — 頁面版面框架

完整頁面的版面元件，每個都是對應路由頁面的主要結構容器。

### `layouts/bootUp/`
登入與機器設定畫面，在使用者尚未驗證身份前顯示。包含 `SignInForm` 與 `MachineConfigForm`。

### `layouts/saleConsole/`
主 POS 操作介面，分為三個面板：
- `item/` — 商品選擇區域（含 `Shortcut`、`Management`、商品清單）
- `salon/` — 美容院服務管理區域
- `summary/` — 附付款流程的購物車摘要（`PayInfo`、`Change`、`EnterBarcode`、edenred pay）

### `layouts/setting/`
門市設定頁面，包含：
- `ExperimentFeature` — 功能旗標
- `PrinterOption` — 印表機設定
- `LocalStorageOperation` — 檢視/清除本地儲存
- `VersionInfo` — 應用程式版本顯示
- `ExtraDialog` — 額外設定對話框

### `layouts/workbench/`
每日作業畫面，包含：
- `WorkSheetDetail` — 工作單任務視圖
- `AssignBeautician` — 美容院美容師指派
- `SelectReportDate` / `ResetChangeTasksConfirm` — 工作台工具元件

## `tasks/` — 功能性對話框

Task 元件是**針對特定功能的對話框或抽屜**，由版面中的使用者操作觸發開啟。它們遵循一致的結構：每個 task 資料夾包含 `index.tsx`（元件本體）、`interface.ts`（props/state 型別定義），通常還有 `style.ts`。

### `tasks/pos/` — POS 任務

主要任務：

| Task | 功能 |
|------|----------|
| `cancelSale` | 取消進行中或已完成的銷售 |
| `coupon` | 套用或管理折扣優惠券 |
| `exchangeSale` | 換貨（針對已售出的商品） |
| `freebie` | 將促銷贈品加入購物車 |
| `fullReturn` / `partialReturn` | 全額退貨或部分退貨 |
| `importDeliveryOrder` | 匯入 FoodPanda / 外送訂單 |
| `itemDetail` | 查看商品詳細資訊 |
| `ItemPriceChange` / `ItemPromotionPriceChange` | 手動或促銷價格覆寫 |
| `orderDiscount` | 套用整筆訂單折扣 |
| `pointPromotion` | 會員點數兌換 |
| `priceSearch` | 查詢商品價格 |
| `printItemReceipt` | 列印單品收據 |
| `promotionItemBatchPrintV2` | 批量列印促銷價格標籤 |
| `sailorStoreCounter` | 加盟門市每日營業計數器 |
| `salesTarget` | 顯示與追蹤銷售目標 |
| `stashSaleBoard` / `stashSaleList` | 暫存/取回進行中的銷售 |
| `addItemsWithAmount` | 以指定數量將商品加入購物車 |
| `addon` | 加購項目管理 |
| `clearAll` | 清除所有購物車項目 |
| `corporatePrintPriceCard` | 企業版價格標籤列印 |
| `customerOrderSearchOrWithdraw` | 搜尋或取消客戶線上訂單 |
| `maPartialExchangeItemTable` | MA 部分換貨項目表格 |
| `newStorePriceCard` | 新門市價格標籤列印 |
| `partialSearchOrExchange` | 部分銷售搜尋或換貨 |
| `pocketPromotion` | 口袋促銷管理 |
| `promotionItemPriceChange` | 促銷商品價格變更 |
| `shelfPrint` | 貨架標籤列印 |
| `sumReport` | 彙總報表 |
| `viewCustomerOrderDetail` | 查看客戶訂單詳情 |
| `voucherExchange` | 禮券兌換 |

### `tasks/salonPos/` — 美容院 POS 任務

| Task | 功能 |
|------|----------|
| `addOrEditServiceItem` | 新增/編輯美容服務項目 |
| `packageServiceItemReturn` | 退還套裝服務項目 |
| `salonClearAll` | 清除所有美容院購物車項目 |
| `salonFullReturn` | 美容院服務全額退款 |
| `salonPromotionList` | 查看可用的美容院促銷活動 |
| `salonSalesTarget` | 美容院員工銷售目標 |
| `salonSumReport` | 美容院彙總報表 |
| `unpaidSaleSearch` | 搜尋未付款的美容院銷售 |
| `workSheetList` | 工作單清單管理 |

### `tasks/common/` — 共用任務

POS 與工作台共同使用：

| Task | 功能 |
|------|----------|
| `saleSearch` | 搜尋歷史銷售紀錄 |
| `dailyReport` | 每日銷售報表 |
| `openingEntry` / `closingEntry` | 開收銀機/關收銀機 |
| `depositPoint` | 會員儲值點數 |
| `memberPointUsage` | 查看會員點數使用紀錄 |
| `expenditureApplication` / `expenditureList` | 零用金管理 |
| `importInvoiceInfo` | 匯入電子發票資料 |
| `petParkCouponUsage` | 寵物公園優惠券兌換追蹤 |
| `editOpeningEntryLog` | 編輯開機紀錄 |
| `batchPrint` | 批量列印標籤或收據 |

---
