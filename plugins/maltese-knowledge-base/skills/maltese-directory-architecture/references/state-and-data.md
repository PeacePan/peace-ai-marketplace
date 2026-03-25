# 狀態管理與資料層

## 架構概覽

```
Browser
  └── React Context (src/contexts/)   ← 跨元件狀態 + GQL 輸入輸出
        ├── reads from IndexedDB (idb contexts)
        ├── reads/writes via GraphQL (src/graphql/)
        └── fires workers (src/workers/)

src/models/     ← TypeScript interfaces matching backend DB schema
src/hooks/      ← Reusable React hooks consuming contexts or devices
src/utils/contexts/  ← Pure business logic helpers used by contexts
```

## `src/contexts/` — React Contexts

每個檔案都是以 `myFuncCompFactory`（來自 `src/libs/` 的 StateDecorator 包裝器）建立的 Context。Context 負責持有主要的跨元件狀態，並處理所有資料變更操作。

**設計規則（來自 `src/contexts/README.md`）：**
- `[idb]` 標記 = 以 IndexedDB 作為查詢目標，而非 GQL
- 公開 API：資料列表使用 GQL body + lines 慣例；mutations 使用應用層級的 action 介面
- `uiCtrller` 是特殊 Context，僅控制對話框／抽屜的開關狀態，不處理資料

### Contexts 列表

> 完整目錄包含 40+ 個 Context 檔案，以下為精選參考清單；請執行 `ls src/contexts/` 查看完整列表。

| Context 檔案 | 領域 | 備註 |
|-------------|--------|-------|
| `posSale.ts` | 當前及歷史銷售 | 所有銷售操作的核心 |
| `posMember.ts` | 會員查詢與操作 | 包含 idb 快取 |
| `posPromotion.tsx` | 有效促銷、贈品、訂單折扣 | 以 idb 為後端 |
| `item.ts` | 商品資料 | 以 idb 為後端 |
| `itemList.tsx` | 目前顯示的商品列表 | |
| `checkoutInfo.ts` | 結帳狀態（付款、發票資訊）| |
| `posReturn.ts` | 退貨／退款狀態 | |
| `customerOrder.ts` | 顧客線上訂單 | |
| `expenditure.ts` | 零用金支出 | |
| `uiCtrller.tsx` | 對話框／抽屜顯示控制 | 無資料，僅 UI 狀態 |
| `general.ts` | 共用工具（獎勵取得等）| |
| `posServiceSale.ts` | 沙龍服務銷售狀態 | |
| `posServiceReturn.ts` | 沙龍服務退貨 | |
| `task.ts` | 背景任務狀態 | |
| `user.tsx` | 已登入的員工 | 以 idb 為後端 |
| `bootUp.ts` | 應用程式啟動初始化狀態（機器設定、登入）| |
| `foodpanda.ts` | FoodPanda 外送訂單狀態 | |
| `itemPriceChange.ts` | 商品價格變更狀態 | |
| `itemPromotionPrice.ts` | 促銷商品價格狀態 | |
| `itemReceipt.ts` | 商品收據狀態 | |
| `itemSalesTarget.ts` | 商品銷售目標狀態 | |
| `maPartialOrder.ts` | MA（Magento）部分訂單狀態 | |
| `maShopOrder.ts` | MA Shop 線上訂單狀態 | |
| `pointPromotion.ts` | 會員點數促銷狀態 | |
| `posCoupon.ts` | POS 折價券狀態 | |
| `posDeposit.ts` | 會員儲值狀態 | |
| `posFreebie.tsx` | 促銷贈品狀態 | |
| `posItemCollection.ts` | 商品集合狀態 | |
| `posMallSetting.tsx` | POS 商場設定狀態 | |
| `posOrderDiscount.tsx` | 訂單層級折扣狀態 | |
| `posPet.tsx` | 寵物資訊狀態 | |
| `posPrintableCoupon.ts` | 可列印折價券狀態 | |
| `posReport.tsx` | POS 報表狀態 | |
| `posSalonCoupon.ts` | 沙龍折價券狀態 | |
| `posSalonPromotion.tsx` | 沙龍促銷狀態 | |
| `posShelf.ts` | 貨架／價格標籤狀態 | |
| `posShortcut.ts` | POS 快捷鍵狀態 | |
| `posVoucher.ts` | 禮券狀態 | |
| `serviceItem.ts` | 沙龍服務項目資料 | |
| `serviceItemList.ts` | 沙龍服務項目列表 | |

> **新增 Context：** 請以 `src/contexts/_template.ts` 作為起始範本。

### `myFuncCompFactory` 模式（來自 `src/libs/README.md`）

在 Maltese 中建立有狀態元件的標準方式。它包裝 `state-decorator`，將狀態／actions 與視圖邏輯分離。

主要慣例：
- `getInitialState` — 產生初始狀態
- `actionsImpl` — 所有狀態變更；純狀態 actions 命名為 `setXXX`，複雜 actions 命名為 `handleXXX`
- `abortableActions` — 支援取消的非同步 actions 列表
- `view` — 渲染函式，通常抽取至同層級的元件
- 若設定 `contextlize: true`，元件也會建立一個 React Context，並回傳 `[ProviderComponent, ReactContext]`

## `src/models/` — 資料模型

每個模型檔案對應一個後端資料庫資料表（來自 `src/models/README.md`）：
- **Body type** — 標頭欄位（例如 `PosMemberBody`）
- **Lines type** — 明細欄位（例如 `PosMemberLines`）
- **Record type** — 完整 GQL 查詢記錄（例如 `PosMember`）
- 相關的子型別與 enum 同置於同一檔案

模型檔案以資料表命名：`posSale.ts`、`posMember.ts`、`posPromotion.ts` 等（共 50+ 個模型檔案）。

## `src/graphql/` — GraphQL 層

Maltese 使用自訂 GraphQL schema（`@norwegianForest*` 共用函式庫），而非標準公開 API。

| 檔案 | 用途 |
|------|---------|
| `apolloClient.ts` | Apollo Client 初始化 |
| `query.ts` | `findMalteseRecord`、`chunkFindMalteseRecord`、`optimizedChunkFindMalteseRecord` |
| `mutation.tsx` | `mutateMalteseRecord`、`call`、`execute` |
| `schema.ts` | GraphQL schema 型別 |
| `sign.ts` | 已驗證 GQL 呼叫的請求簽署 |
| `interface.ts` | GQL 記錄、篩選條件、排序的 TypeScript 型別 |

來自 `src/graphql/README.md`：
- 查詢資料表時，一律先呼叫 `gqlFindTableRecord`（取得資料表定義），再呼叫 `gqlFindRecord`
- mutations 中的數字欄位必須包裝為 `{ _set: number; _incr: number }` FieldUpdate 物件
- 明細層級 ReadWrite 規則（`READ` / `FULL` / `UPDATE` / `PULL` / `PUSH`）控制明細的新增、刪除或修改權限
- 欄位層級 ReadWrite 規則（`READ` / `UPDATE` / `INSERT`）控制個別欄位的讀取、更新或插入權限

## `src/hooks/` — 自訂 Hooks

| Hook | 用途 |
|------|---------|
| `useScene.ts` | 場景／頁面生命週期管理 |
| `useMagentoAPI.ts` | Magento REST API 整合 Hook |
| `useInvoiceDevice.ts` | 電子發票裝置通訊 |
| `useTaishinOnePay.ts` | 台新 Pay 終端機 Hook |
| `useBroadcastChannel.ts` | 跨分頁訊息傳遞 |
| `useItemCollection.ts` | 商品集合狀態 |
| `usePollingWithActions.ts` | 支援中止的輪詢 |
| `useUpdateAndCheckPosVersion.ts` | 版本檢查輪詢 |
| `usePromiseifyWorker.ts` | 將 Web Worker 包裝為 Promise |
| `useAsync.ts` | 非同步 action 狀態管理 |
| `useDebounceAction.ts` / `useThrottle.ts` | 頻率限制 Hooks |
| `useKeyboardActions.ts` | 鍵盤快捷鍵綁定 |
| `useInfiniteScroll.ts` | 列表無限捲動 |
| `useAbortMyFCActionWhenUnmount.ts` | 元件卸載時中止所有進行中的 myFuncComp actions |
| `useBindFocus.ts` | 將焦點行為綁定至元素 |
| `useGoogleChat.ts` | Google Chat 通知 Hook |
| `useInfiniteScrollParent.ts` | 父層級無限捲動容器 |
| `useInterval.ts` | 具自動清除功能的 SetInterval |
| `useIsMounted.ts` | 追蹤元件是否仍已掛載 |
| `useIsomorphicLayoutEffect.ts` | 可在 SSR 環境運作的 useLayoutEffect |
| `useOpenItemDetailDialog.tsx` | 以程式方式開啟商品詳情對話框 |
| `usePrintableCoupon.ts` | 可列印折價券管理 Hook |
| `useSafeState.ts` | 卸載後忽略更新的 useState |
| `useScroll.ts` | 捲動位置追蹤 |

## `src/workers/` — Web Workers

| Worker | 用途 |
|--------|---------|
| `webWorker/calcBestPosPromotion.worker.ts` | 在主執行緒外執行促銷組合演算法 |
| `sharedWorker/` | 跨分頁共用 Worker，用於即時狀態同步 |
| `createPromiseifyWorker.ts` | `usePromiseifyWorker` Hook 使用的工廠函式，將 Worker 包裝為 Promise |

## `src/algorithm/` — 促銷引擎

純 TypeScript 演算法，無 React／瀏覽器依賴：
- `bucketCombination` — 計算購物車商品所有可能的促銷組合
- `calcBestAnswer` — 選取最高優惠的促銷組合

這些演算法由 Web Worker 及單元測試直接使用。

此目錄另包含：
- `class.ts` — `BucketItemFilter` — 具有記憶化功能的商品判斷類別，用於加速 bucket 篩選
- `interface.ts` — `CombinationItem` 型別及促銷引擎相關介面

## `src/utils/contexts/` — Context 業務邏輯輔助函式

從 Context 中抽取出來的純工具函式，用於實作複雜業務規則，以利測試：

| 檔案 | 領域 |
|------|--------|
| `checkout.ts` | 發票 ID 產生、付款計算 |
| `posSale.ts` | 未付款銷售單產生 |
| `promotion.ts` | 促銷篩選／套用邏輯 |
| `return.ts` | 退貨商品計算 |
| `payments.ts` | 付款方式輔助函式 |
| `freebie.ts` | 贈品資格判斷輔助函式 |
| `itemList.ts` | 商品列表操作 |
| `serviceSale.ts` | 沙龍服務銷售輔助函式 |
| `conditions.ts` | GQL 條件建構器 |
| `filters.ts` | 資料篩選輔助函式 |
| `petParkCoupon.ts` | 寵物樂園折價券輔助函式 |
| `posCouponCode.ts` | POS 折價券代碼輔助函式 |
| `printableCoupon.ts` | 可列印折價券輔助函式 |
| `serviceList.ts` | 服務列表輔助函式 |

---
