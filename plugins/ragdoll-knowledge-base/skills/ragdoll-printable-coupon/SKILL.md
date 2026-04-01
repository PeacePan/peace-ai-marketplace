---
name: ragdoll-printable-coupon
description: >
  Ragdoll 小白單（紙本優惠券）列印功能完整知識庫。涵蓋結帳後的資格篩選邏輯、
  pos_printable_coupon 資料表結構、sync 同步機制、IPC 列印呼叫，
  以及 Next.js 層的 filterAndBuildPrintableCoupons 工具函式與 useSale action。

  以下情況必須參考此文件再動手：
  - 修改或擴充小白單優惠券的篩選條件（滿額/門市群組/目標消費者/料件篩選）
  - 修改小白單的列印格式（topText、barcodeContent、titles、memo）
  - 修改 pos_printable_coupon SQLite schema 或 sync config
  - 在結帳流程中調整小白單列印的觸發時機
  - 撰寫小白單相關的單元測試或整合測試
  - 理解 IPC 資料流（Next.js store → Electron 主進程 → 發票機）
---

# Ragdoll 小白單（紙本優惠券）列印知識庫

## 業務概覽

小白單（紙本優惠券）是結帳完成後由發票機自動列印的紙本優惠券，用於吸引顧客下次回購。

**觸發時機**：結帳流程完成發票列印後，自動依消費資格篩選並列印（最多 3 張）。

**列印條件**：
- 消費滿額（saleTotal）：銷售總額必須達到設定門檻
- 門市群組（storeGroups）：目前門市必須在允許群組內
- 目標消費者（targetConsumer）：MEMBER / NON_MEMBER / ALL
- 料件篩選器（filters）：購買商品須符合設定的料件條件
- 有效期（startAt / endAt）：由 sync config 在查詢時過濾，確保只同步當日有效資料

**不支援的情境（此版本）**：
- 有 `couponEventName` 的優惠券（寵物公園活動券，需要 PetParkCouponEvent 資料）
- 首次消費（onlyFirstConsumption）、累積消費（accumulationSaleTotal）、列印上限（printLimit）的會員歷史查詢

---

## 架構概覽

```
[結帳完成]
  ↓
[checkout-button.tsx]
  print-invoice 步驟完成後
  → saleActions.printPrintableCouponsForSale()
      ↓
  [use-sale.ts: printPrintableCouponsForSale action]
    1. getPosConfig() → 取得門市名稱
    2. filterAndBuildPrintableCoupons() → 篩選 + 組裝 PrintableCouponInput[]
    3. printInvoice({ mode: '優惠券', coupons: [...] }) → IPC
        ↓
    [ragdollAPI.devices.invoice.print]
    → Electron 主進程 → 發票機 HTTP POST /print
```

**資料同步流程（背景排程）**：
```
GraphQL posprintablecoupon → sync config → pos_printable_coupon（本地 SQLite）
```

---

## 關鍵檔案路徑

| 功能 | 路徑 |
|---|---|
| **SQLite 主表 Schema** | `electron/main/database/tables/readonly/pos_printable_coupon/pos_printable_coupon.sql` |
| **SQLite 門市群組表身** | `electron/main/database/tables/readonly/pos_printable_coupon/pos_printable_coupon_lines_store_groups.sql` |
| **SQLite 料件篩選器表身** | `electron/main/database/tables/readonly/pos_printable_coupon/pos_printable_coupon_lines_filters.sql` |
| **SQLite 列印備註表身** | `electron/main/database/tables/readonly/pos_printable_coupon/pos_printable_coupon_lines_coupon_print_memo.sql` |
| **TypeScript 型別定義** | `electron/main/database/tables/readonly/pos_printable_coupon/index.ts` |
| **同步設定** | `electron/main/jobs/sync-data/configs/posprintablecoupon.ts` |
| **IPC 型別（PrintableCouponInput）** | `electron/main/types/ipc-devices.ts` |
| **篩選工具函式** | `next/lib/utils/coupon/filter.ts` |
| **useSale action** | `next/lib/stores/checkout/sale/use-sale.ts`（`printPrintableCouponsForSale`） |
| **結帳按鈕整合** | `next/app/summary/components/checkout-button.tsx` |

---

## 資料庫 Schema（`pos_printable_coupon`）

### 主表欄位

| 欄位 | 型別 | 說明 |
|---|---|---|
| `name` | TEXT PK | 優惠券編號 |
| `print_channel` | TEXT | 列印適用渠道 |
| `display_name` | TEXT | 活動名稱 |
| `cooperation_partner` | TEXT | 配合廠商 |
| `priority` | TEXT | 優先級（用於排序，升冪） |
| `start_at` | INTEGER | 起始日期（Unix timestamp） |
| `end_at` | INTEGER | 結束日期（Unix timestamp） |
| `sale_total` | REAL | 消費滿額門檻（0 表示無限制） |
| `only_first_consumption` | INTEGER | 限定首次消費（0/1） |
| `accumulation_start_at` | INTEGER | 期間累積消費開始日期 |
| `accumulation_end_at` | INTEGER | 期間累積消費結束日期 |
| `accumulation_sale_total` | REAL | 期間累積消費金額門檻 |
| `target_consumer` | TEXT | 目標消費者（MEMBER/NON_MEMBER/ALL） |
| `coupon_event_name` | TEXT | 優惠券活動編號（此版本略過） |
| `print_limit` | INTEGER | 列印上限（此版本略過） |
| `need_collect` | INTEGER | 是否回收（0/1） |
| `coupon_top` | TEXT | 優惠券頂部文字 |
| `barcode_content` | TEXT | 條碼內容 |
| `barcode_type` | TEXT | 條碼樣式（enum） |
| `title1/2/3` | TEXT | 標題文字（各三組） |
| `title1/2/3_bold` | INTEGER | 粗體（0/1） |
| `title1/2/3_font_size` | TEXT | 字體大小（SMALL/MEDIUM/LARGE） |
| `title1/2/3_align` | TEXT | 對齊方式（LEFT/CENTER/RIGHT） |
| `expiration_start_at` | INTEGER | 有效期限開始日期（Unix timestamp） |
| `expiration_end_at` | INTEGER | 有效期限結束日期（Unix timestamp） |

### 表身

- `pos_printable_coupon_lines_store_groups`：門市群組（groupName、effect）
- `pos_printable_coupon_lines_filters`：料件篩選器（filterName、name、brand、category 等）
- `pos_printable_coupon_lines_coupon_print_memo`：列印備註（text）

---

## Sync Config（`posprintablecoupon.ts`）

```typescript
{
  localTable: 'pos_printable_coupon',
  remoteTable: 'posprintablecoupon',
  getFindParams: async () => ({
    filters: [{ body: { startAt: { $lte: today }, endAt: { $gt: today } } }],
    selects: ['body', 'lines.storeGroups', 'lines.filters', 'lines.couponPrintMemo'],
  }),
  recordsFilter: 依門市群組篩選,
  recordMapper: 遠端 Date → 本地 Unix timestamp、boolean → 0/1,
  recordCleaner: 清除不適用當前門市的資料,
}
```

---

## 篩選邏輯（`filter.ts`）

`filterAndBuildPrintableCoupons()` 執行步驟：

1. **查詢**：`ragdollAPI.db.list('pos_printable_coupon')` + `ragdollAPI.db.list('pos_store_group')`
2. **取得料件資訊**：`fetchItemsInformation(itemNames)`
3. **篩選**（每張優惠券依序檢查）：
   - 有 `couponEventName` → **略過**
   - `saleTotal > total` → **排除**
   - `filterByStoreGroups(storeName, storeGroups, posStoreGroupMap)` → **排除**
   - 有會員 + `targetConsumer === 'NON_MEMBER'` → **排除**
   - 無會員 + `targetConsumer === 'MEMBER'` → **排除**
   - `filters` 有值但無商品命中 `isItemInFilterCondition` → **排除**
4. **排序**：依 `priority` 升冪
5. **截取**：最多 3 筆
6. **組裝**：轉換為 `PrintableCouponInput[]`

---

## 列印 IPC 格式

```typescript
// InvoicePrintRequest（mode='優惠券' 時）
{
  mode: '優惠券',
  coupons: PrintableCouponInput[],
  // 以下欄位對優惠券模式無作用，但 type 要求必須傳
  total: 0, tax: 0, items: [], payments: [],
  storeName: 'PRINT_COUPON', configNo: 'PRINT_COUPON',
  invoiceNo: '', randomCode: '', buyerTaxCode: '', sellerTaxCode: '',
  createdAt: new Date().toISOString(),
  shouldOpenCashDrawer: false,
}

// PrintableCouponInput 結構
{
  topText: string,          // couponTop
  barcodeType: string,      // enum
  barcodeContent: string,
  titles: PrintableCouponTitle[],  // 最多 3 組（title1/2/3）
  expirationStartAt: Date,  // Unix timestamp 轉換
  expirationEndAt: Date,
  memo: string[],           // couponPrintMemo.text 陣列
}
```

---

## 結帳流程整合

`checkout-button.tsx` 的步驟順序：

```
1. create-sale    建立銷售紀錄
2. print-invoice  列印發票
2.5. print-coupon 列印小白單（發票機在線才執行，失敗不阻斷後續）
3. open-drawer    開啟錢櫃
4. clear-cart     清除購物車
```

**設計原則**：
- 小白單列印失敗只更新步驟狀態為 `error`，不 `return` 或 `throw`
- 發票機離線時與 print-invoice 同樣設為 `skipped`
- `couponsPrinted: boolean` 記錄於 `SaleData`（`use-sale.ts`）

---

## 移植來源

此功能移植自 Maltese 的 `createPrintableCouponProcessor`：
- `src/utils/contexts/printableCoupon.ts`
- `src/contexts/posPrintableCoupon.ts`
- `src/models/posPrintableCoupon.ts`

**主要簡化點**：
- 略過 `couponEventName`（PetParkCouponEvent 不在 Ragdoll 本地資料庫）
- 略過 `onlyFirstConsumption`、`accumulationSaleTotal`、`printLimit`（需會員歷史銷售記錄，Ragdoll 本地無此資料）
- 略過員購通路排除邏輯（Ragdoll 目前只有門市通路）
