# Task-based 測試組織模式

## 架構概念

Maltese E2E 測試採用 **Entry → Task** 兩層組織模式：

```
entries/              # 測試進入點（Jest project 指向這裡）
├── 000_dataInit      # 資料初始化（獨立 config 執行）
├── 001_posMain       # → tasks/posMain/*
├── 002_posETC        # → tasks/posETC/*
├── 003_salon         # → tasks/salon/*
└── 004_workbench     # → tasks/workbench/* + tasks/common/* + tasks/salon/*

tasks/                # 測試邏輯實作
├── common/           # 共用 task（bootUp、report、user、saleSearch、utils）
├── posMain/          # POS 主要功能 tasks（14 個）
├── posETC/           # POS 附加功能 tasks（12 個）
├── salon/            # 美容服務 tasks（11 個）
└── workbench/        # 美容工作臺 tasks（3 個）
```

## Entry 與 Task 的關係

**Entry file** 是 Jest 的測試進入點，負責：
1. 呼叫 `setupBrowserPage(page, workerId)` 設定瀏覽器
2. 呼叫 `bootUpTask(config)` 執行登入
3. 依序呼叫各 task function

**Task function** 是一個回傳 `void` 的函式，內部包含 `describe()` 和 `it()` 項目：

```typescript
// tasks/posMain/001_checkout.test.ts
export const posCheckoutTask = () => {
  describe('一般商品結帳測試', () => {
    // 可選：generateBatchInsertTestTask 建立測試資料
    generateBatchInsertTestTask([...]);

    it('結帳流程步驟 1', async () => { /* ... */ });
    it('結帳流程步驟 2', async () => { /* ... */ });
  });
};
```

**關鍵特性：**
- Task function 被 entry file **同步呼叫**，Jest 按呼叫順序注冊 `describe`/`it`
- 所有 task 共享同一個全域 `page`（同一個 browser 實體）
- 前一個 task 的頁面狀態會保留到下一個 task
- `jasmine-fail-fast` 確保一個 test 失敗即跳過後續所有 test

---

## 4 個套件的完整 Task 呼叫清單

### Suite 1: posMain（worker 1, 電子發票）

**Entry:** `entries/001_posMain.test.ts`
**Config:** `cMockEGUITypePosConfig`

| 順序 | Task Function | 來源檔案 | 說明 |
|------|--------------|----------|------|
| 1 | `setupBrowserPage(page, 1)` | `utils.ts` | 設定 browser worker 1 |
| 2 | `bootUpTask(config)` | `common/001_bootUp.test.ts` | 登入 + 選擇門市機台 |
| 3 | `checkoutInitTask()` | `posMain/000_init.test.ts` | 初始化結帳測試資料 |
| 4 | `enterEntryTask('item')` | `common/utils.test.ts` | 進入商品銷售介面 |
| 5 | `backToMainMenuTask('item', member)` | `common/utils.test.ts` | 測試回到主選單 |
| 6 | `openReportTask(config)` | `common/002_posReport.test.ts` | 開啟報表 |
| 7 | `posCheckoutTask()` | `posMain/001_checkout.test.ts` | 一般商品結帳 |
| 8 | `stashSaleTask()` | `posMain/002_stashSale.test.ts` | 暫存銷售 |
| 9 | `deletePaymentTask()` | `posMain/003_deletePayment.test.ts` | 刪除付款 |
| 10 | `fullReturnTask(config)` | `posMain/004_fullReturn.test.ts` | 全額退貨 |
| 11 | `pocketPromotionTask()` | `posMain/005_pocketPromotion.test.ts` | 口袋促銷 |
| 12 | `promotionTask()` | `posMain/006_promotion.test.ts` | 促銷 |
| 13 | `importFastBuyOrder()` | `posMain/007_importFastBuyOrder.test.ts` | 匯入快速訂單 |
| 14 | `orderDiscountTask()` | `posMain/008_orderDiscount.test.ts` | 訂單折扣 |
| 15 | `exchangeSaleTask()` | `posMain/009_exchangeSale.test.ts` | 換貨 |
| 16 | `petparkCouponTask()` | `posMain/010_petparkCoupon.test.ts` | PetPark 優惠券 |
| 17 | `partialOrderTask()` | `posMain/011_partialOrder.test.ts` | 部分訂單 |
| 18 | `customerOrderHistoryTask()` | `posMain/012_customerOrderHistory.test.ts` | 顧客訂單歷史 |
| 19 | `memberPromotionTask()` | `posMain/013_memberPromotion.test.ts` | 會員促銷 |

---

### Suite 2: posETC（worker 2, 收據）

**Entry:** `entries/002_posETC.test.ts`
**Config:** `cMockReceiptTypePosConfig`

| 順序 | Task Function | 來源檔案 | 說明 |
|------|--------------|----------|------|
| 1 | `setupBrowserPage(page, 2)` | `utils.ts` | 設定 browser worker 2 |
| 2 | `bootUpTask(config)` | `common/001_bootUp.test.ts` | 登入 + 選擇門市機台 |
| 3 | `checkoutInitTask()` | `posETC/000_init.test.ts` | 初始化 posETC 測試資料 |
| 4 | `enterEntryTask('item')` | `common/utils.test.ts` | 進入商品銷售介面 |
| 5 | `openReportTask(config)` | `common/002_posReport.test.ts` | 開啟報表 |
| 6 | `depositPointTask()` | `posETC/001_depositPoint.test.ts` | 儲值點數 |
| 7 | `fullReturnTask(config)` | **posMain**/004_fullReturn.test.ts | 全額退貨（**借用 posMain 的 task**） |
| 8 | `addonTask()` | `posETC/002_addon.test.ts` | 加購 |
| 9 | `freebieTask()` | `posETC/003_freebie.test.ts` | 贈品 |
| 10 | `voucherExchangeTask()` | `posETC/004_voucherExchange.test.ts` | 兌換券 |
| 11 | `couponTask()` | `posETC/005_coupon.test.ts` | 優惠券 |
| 12 | `adjustPriceTask()` | `posETC/006_adjustPrice.test.ts` | 調整價格 |
| 13 | `openPriceItemTask()` | `posETC/007_openPriceItem.test.ts` | 開放定價商品 |
| 14 | `priceSearchTask()` | `posETC/008_priceSearch.test.ts` | 價格查詢 |
| 15 | `saleSearchTask('item')` | **common**/005_saleSearch.test.ts | 銷售查詢（**共用 task**） |
| 16 | `disableUser()` | `posETC/010_disableUser.test.ts` | 停用使用者 |
| 17 | `openCloseReportTask()` | `posETC/011_openCloseReport.test.ts` | 開收班結帳 |

**注意：** `tasks/posETC/009_shelf.test.ts` 存在但未被任何 entry 引用，為 dead code。

---

### Suite 3: salon（worker 3, 電子發票）

**Entry:** `entries/003_salon.test.ts`
**Config:** `cMockEGUITypeSalonConfig`

| 順序 | Task Function | 來源檔案 | 說明 |
|------|--------------|----------|------|
| 1 | `setupBrowserPage(page, 3)` | `utils.ts` | 設定 browser worker 3 |
| 2 | `bootUpTask(config)` | `common/001_bootUp.test.ts` | 登入 + 選擇門市機台 |
| 3 | `salonInitTask()` | `salon/000_init.test.ts` | 初始化美容測試資料 |
| 4 | `enterEntryTask('salon')` | `common/utils.test.ts` | 進入美容服務介面 |
| 5 | `backToMainMenuTask('salon', member)` | `common/utils.test.ts` | 測試回到主選單 |
| 6 | `openReportTask(config)` | `common/002_posReport.test.ts` | 開啟報表 |
| 7 | `posMemberTask(member)` | `salon/001_posMember.test.ts` | 會員操作 |
| 8 | `posPetTask(member, pet)` | `salon/002_posPet.test.ts` | 寵物操作 |
| 9 | `saleSearchTask('salon', member)` | **common**/005_saleSearch.test.ts | 銷售查詢（**共用 task**） |
| 10 | `addServiceItemTask()` | `salon/003_addServiceItem.test.ts` | 加入美容服務項目 |
| 11 | `salonPromotionTask()` | `salon/004_promotion.test.ts` | 美容促銷 |
| 12 | `deletePaymentTask()` | `salon/008_deletePayment.test.ts` | 刪除付款（**在結帳前**） |
| 13 | `salonCheckoutTask()` | `salon/005_checkout.test.ts` | 美容結帳 |
| 14 | `unpaidServiceSaleTask()` | `salon/006_unpaidServiceSale.test.ts` | 未付款服務銷售 |
| 15 | `depositPointTask()` | `salon/009_depositPoint.test.ts` | 儲值點數 |
| 16 | `salonReturnTask(config, member)` | `salon/007_return.test.ts` | 美容退貨 |
| 17 | `openCloseReportTask()` | `salon/010_openCloseReport.test.ts` | 開收班結帳 |

---

### Suite 4: workbench（worker 4, 收據）

**Entry:** `entries/004_workbench.test.ts`
**Config:** `cMockWorkbenchConfig`（工作臺階段）→ `cMockReceiptTypeSalonConfig`（美容階段）

此套件較特殊，中途會切換機台設定：

| 順序 | Task Function | 來源檔案 | 說明 |
|------|--------------|----------|------|
| 1 | `setupBrowserPage(page, 4)` | `utils.ts` | 設定 browser worker 4 |
| 2 | `bootUpTask(workbenchConfig)` | `common/001_bootUp.test.ts` | 登入（工作臺機台） |
| 3 | `workbenchInitTask()` | `workbench/000_init.test.ts` | 初始化工作臺資料 |
| 4 | `enterEntryTask('workbench')` | `common/utils.test.ts` | 進入工作臺介面 |
| 5 | `finishWorksheetTask()` | `workbench/001_finishWorksheet.test.ts` | 完成工作單 |
| — | **切換機台** | （inline code） | `localStorage.setItem` 切換為美容收據機台 |
| 6 | `enterEntryTask('salon', false)` | `common/utils.test.ts` | 進入美容介面（不自動設定銷售員） |
| 7 | `userTask()` | **common**/003_user.test.ts | 使用者測試 |
| 8 | `openReportTask(salonConfig)` | **common**/002_posReport.test.ts | 開啟報表 |
| 9 | `editOpeningEntryLogTask()` | **common**/002_posReport.test.ts | 編輯開班紀錄 |
| 10 | `expenditureApplicationTask()` | **common**/004_expenditureApplication.test.ts | 零用金申請 |
| 11 | `posPetTask(member, pet)` | **salon**/002_posPet.test.ts | 寵物操作 |
| 12 | `workSheetTask()` | `workbench/002_workSheet.test.ts` | 工作單操作 |
| 13 | `salonReturnTask(salonConfig, member)` | **salon**/007_return.test.ts | 美容退貨 |

**特殊點：** workbench 是唯一一個在執行途中切換機台設定的套件，且大量引用 `common/` 和 `salon/` 的 task functions。

---

## 共用 Task Functions（tasks/common/）

| 檔案 | 輸出的 Task Functions | 被哪些 Suite 使用 |
|------|----------------------|-----------------|
| `001_bootUp.test.ts` | `bootUpTask(config)` | 全部 4 個 |
| `002_posReport.test.ts` | `openReportTask(config)`, `editOpeningEntryLogTask()` | posMain, posETC, salon, workbench |
| `003_user.test.ts` | `userTask()` | workbench |
| `004_expenditureApplication.test.ts` | `expenditureApplicationTask()` | workbench |
| `005_saleSearch.test.ts` | `saleSearchTask(entry, member?)` | posETC, salon |
| `utils.test.ts` | `enterEntryTask(entry)`, `backToMainMenuTask(entry, member)`, `addItemToList(items)`, `enterCheckout(page, opts)`, `selectPosMember(member)`, `setPayInfo(info)`, `setupSaler(page, saler)`, `clearAll()`, `generateBatchInsertTestTask(params)`, `findItemTableRows()`, `waitItemListIdle()` | 全部 |

---

## Mock 資料（tests/e2e/mock/）

25 個 TypeScript 檔案，每個匯出 `cMock*` 常數：

| 檔案 | 主要匯出 | 說明 |
|------|---------|------|
| `item.ts` | `commonItemBody`, `commonItemLines`, `cMockPromotionItems` 等 | 商品資料 |
| `posconfig.ts` | `cMockEGUITypePosConfig`, `cMockReceiptTypePosConfig`, `cMockEGUITypeSalonConfig`, `cMockReceiptTypeSalonConfig`, `cMockWorkbenchConfig`, `cMockOpenCloseEntryConfig` | 6 種機台設定 |
| `posmember.ts` | `cMockPosMember`, `cMockSalonMember`, `cMockWorkbenchMember` | 會員資料 |
| `user.ts` | `cMockEmployee`, `cMockSaler`, `cMockRoles` | 使用者 + 角色 |
| `posstore.ts` | `cMockPosStore`, `cMockOpenCloseEntryPosStore` | 門市資料 |
| `promotion.ts` | `cMockPromotions` | 促銷規則 |
| 其餘 | addon, customerOrder, freebie, partialItem, petparkcoupon, poscoupon, posorderdiscount, pospet, pospilot, posservicesale, posstoregroup, posvoucher, salonPromotion, serviceitem, shelf, memberPromotion, packageServiceItem | 各功能測試資料 |

Mock 資料的格式遵循 NorwegianForest 的 `SafeRecord2` 結構：

```typescript
export const cMockPosMember: PosMember = {
  body: { name: 'testMember...', phone: '0912345678', ... },
  lines: { ... },
};
```

---

## setupBrowserPage 機制

每個 entry 的第一行呼叫 `setupBrowserPage(page, workerId)`，負責：

```typescript
export function setupBrowserPage(page: Page, workerId: number, args = {}) {
  // 1. 載入 dayjs plugins
  dayjs.extend(dayjsSameOrBeforePlugin);
  dayjs.extend(dayjsSameOrAfterPlugin);

  // 2. 啟用 fail-fast（除非 args.noFailFast）
  if (!args.noFailFast) {
    jasmine.getEnv().addReporter(failFast.init());
  }

  // 3. beforeAll: 設定 worker ID + 時區
  beforeAll(async () => {
    process.env.JEST_WORKER_ID = `${workerId}`;
    await page.emulateTimezone('Asia/Taipei');
  });

  // 4. beforeEach: 印出當前測試名稱
  beforeEach(() => {
    console.log(`-> ${expect.getState().currentTestName}`);
  });

  // 5. afterAll: 清理該 worker 的測試資料
  afterAll(async () => {
    await dataArchiver.clean(process.env.JEST_WORKER_ID!);
  });
}
```

**workerId 對應：**
- 1 = posMain
- 2 = posETC
- 3 = salon
- 4 = workbench
