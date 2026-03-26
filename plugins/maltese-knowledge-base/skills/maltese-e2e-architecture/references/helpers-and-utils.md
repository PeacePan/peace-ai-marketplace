# Helpers 與 Utils

所有共用函式集中在 `tests/e2e/utils.ts`（約 740 行），分為兩大類：DOM Helpers 和 GraphQL Helpers。

---

## DOM Helpers

### `clickByText(page, text, selector, options?)`

點擊具有指定文字的元素。**包含 DOM detach 容錯機制**。

```typescript
async function clickByText(
  page: Page,
  text: string,
  selector: ElementHandle | string,
  options?: { timeout?: number; condition?: 'contain' | 'equal' }
): Promise<void>
```

**行為：**
1. 呼叫 `waitForTextInElement()` 等待文字出現
2. 檢查元素是否仍 connected（`elem.isConnected`）
3. 如果 disconnected，等 100ms 後遞迴重試
4. 點擊時若拋出 `Node is detached from document`，fallback 到 `page.evaluate(elem.click())`
5. 最後呼叫 `target.dispose()` 釋放 ElementHandle

**此 detach 容錯很重要：** React re-render 可能在 `waitForSelector` 找到元素後、`click` 前將 DOM 節點替換。

---

### `waitForTextInElement(page, text, selector, options?)`

輪詢等待指定文字出現在 selector 匹配的元素中。

```typescript
async function waitForTextInElement(
  page: Page, text: string, selector: string,
  options?: { timeout?: number; condition?: 'contain' | 'equal' }
): Promise<ElementHandle<Element>>
```

**行為：**
1. 先 `page.waitForSelector(selector)` 確保 selector 存在
2. 每 200ms 輪詢所有匹配的元素
3. 對每個元素同時檢查 textContent 和 `isVisible()`
4. timeout 預設 15 秒

---

### `selectMuiMenuItem(page, { selectSelector, itemValue })`

操作 MUI Select 元件。

```typescript
async function selectMuiMenuItem(page: Page, params: {
  selectSelector: string;  // MUI Select 的 CSS selector
  itemValue: string;       // 要選取的項目值
}): Promise<void>
```

**行為：**
1. 等待 select 元素非 disabled（`:not([aria-disabled=true])`）
2. 點擊 select 觸發下拉
3. 等待 `aria-expanded=true`
4. **等待 300ms**（MUI Popover CSS transform 動畫）
5. 點擊 `.MuiPopover-root .MuiMenuItem-root[data-value="${itemValue}"]`
6. 等待 Popover 關閉
7. **驗證選取結果**：讀取 `<input>` 的 value 確認等於 `itemValue`

**300ms 等待是必要的：** MUI Popover 有 CSS transform 動畫，不等待會導致 Puppeteer 點擊位置計算錯誤。

---

### `getToastMessage(page, args?)`

讀取並關閉 toast 通知。

```typescript
async function getToastMessage(
  targetPage: Page,
  args?: {
    waitting?: boolean;    // 是否等待 toast 出現
    keepMessage?: boolean; // 是否保留 toast 不關閉
    intent?: 'success' | 'error' | 'warning' | 'info';
  }
): Promise<string>
```

**Toast selector：**
- 容器：`[data-testid="muiSnackBar"].MuiSnackbar-anchorOriginTopRight`
- 訊息：`[data-testid="muiAlertMsg"]`
- 帶 intent：`.MuiAlert-color${capitalize(intent)} > .MuiAlert-message`

**重要行為：** 讀取完訊息後會**自動點擊 toast 關閉它**，並等待 snackbar 隱藏。這是為了避免殘留 toast 遮擋後續測試的可點擊區域。如果 `keepMessage: true` 則不關閉。

---

### `expectToastMessage(page, message)`

簡化版：等待 toast 出現 → 驗證包含指定文字 → 自動關閉。

```typescript
async function expectToastMessage(page: Page, message: string): Promise<void> {
  const toastMessage = await getToastMessage(page, { waitting: true });
  expect(toastMessage).toContain(message);
}
```

---

### `scrollToBottom(selector)`

將最近的可捲動父容器捲到底部。**注意：使用全域 `page`。**

```typescript
async function scrollToBottom(selector: string): Promise<void>
```

**行為：**
1. 在 `page.evaluate` 中執行
2. 從目標元素向上尋找最近的可捲動容器（`overflow: auto|scroll` 且 `scrollHeight > clientHeight`）
3. 找不到則使用 `document.body`
4. 遞迴捲動直到到達底部（容忍值 10px）
5. 每次捲動間隔 ~33ms（1000/30）

---

### `waitForTransitionEnd(page, selector)`

等待 CSS transition 結束。

```typescript
async function waitForTransitionEnd(page: Page, selector: string): Promise<void>
```

**行為：** 監聽 `transitionend` 事件，500ms fallback timeout。

---

### `findElementByText(page, text, selector, matchSubString?)`

在指定 selector 的所有元素中尋找文字匹配的元素。

```typescript
async function findElementByText(
  page: Page, text: string, selector: string, matchSubString?: boolean
): Promise<ElementHandle[]>
```

找不到時會 `throw new Error`。

---

### `getElementText(elem)`

取得 ElementHandle 的 `textContent`。

---

### `cloneBrowser(sourcePage, params?)`

複製瀏覽器實體，攜帶 cookie 和 localStorage。用於壓力測試。

```typescript
async function cloneBrowser(
  sourcePage: Page,
  params?: {
    destUrl?: string;
    overwriteLocalStorage?: Record<string, string>;
  }
): Promise<Browser>
```

**行為：**
1. `puppeteer.launch()` 建立新 browser
2. 複製 JavaCat GraphQL 端點的 cookie（HTTPS domain）
3. 複製來源頁面的 cookie 和 localStorage
4. 導航到目標 URL
5. 回傳新 browser 實體

**使用場景：** 壓力測試中快速複製已登入的瀏覽器，不需重新執行登入流程。

---

### `retry(page, action, condition, options?)`

重複執行動作直到目標元素出現。

```typescript
async function retry(
  page: Page,
  action: () => Promise<unknown>,
  condition: string,  // CSS selector
  options?: { timeout?: number; interval?: number }
): Promise<ElementHandle>
```

預設 timeout 30 秒，interval 1 秒。

---

## GraphQL Helpers

所有 GraphQL 操作使用 `cTestGraphQLClient`（Apollo Client，定義在 `consts.ts`）。

### `setupAdminJWT()`

取得管理員 JWT token 並設定到 `cSignRef.jwt`。後續所有 GraphQL 請求會自動帶上此 token。

```typescript
async function setupAdminJWT(): Promise<void>
```

**只會執行一次**（if `cSignRef.jwt` 已有值則 early return）。使用 `E2E_ADMIN_ACCOUNT` 進行 `signin` mutation。

---

### `insertUniAPI(tableName, data, options?)`

通用資料新增。

```typescript
async function insertUniAPI(
  tableName: TableName,
  data: NormRecord2 | NormRecord2[],
  options?: {
    disableRetryWhenNameDuplicate?: boolean;
    archiveAfterTest?: boolean;  // 預設 true
    enableApproval?: boolean;
  }
): Promise<string[]>
```

**行為：**
1. 執行 `insertV2` GraphQL mutation
2. 如果 `enableApproval`，遞迴呼叫 `approveRecordById` 直到簽核通過
3. 如果 `archiveAfterTest: true`（預設），呼叫 `dataArchiver.addLocators()` 註冊待封存資料
4. 名稱重複時（平行測試可能衝突），等待 `(retry + 1) * 5000ms` 後重試一次

**注意：此函式有 side effect** — 會呼叫 `dataArchiver.addLocators()`，意味著它不是純 GraphQL 操作。

---

### `archiveUniAPI(endpoint, data)`

通用資料封存（軟刪除）。

```typescript
async function archiveUniAPI(
  endpoint: TableName,
  data: RecordLocator[]  // [{ keyName: '_id', keyValue: '...' }]
): Promise<string[]>
```

---

### `findBodyValueByLocators(table, locators, fieldNames)`

根據 RecordLocator 查詢指定欄位的值。

```typescript
async function findBodyValueByLocators<R, FieldName>(
  table: TableName,
  locators: RecordLocator | RecordLocator[],
  fieldNames: FieldName[]
): Promise<Pick<R['body'], FieldName> | null>
```

---

### `gqlGetRecord(getArgs)`

透過 keyName/keyValue 取得單筆完整資料（包含 body + lines）。

---

### `callFunction(tableName, functionName, argument?)`

呼叫 JavaCat 表格的 function script。

---

### `archiveByBodyRecordAPI(endpoint, args)`

根據 body 欄位值查詢後封存，用於未知 `_id` 的場景。

---

### `updateMockRecordValueById(table, mockRecords, insertedIds, fieldNames)`

新增資料後，將伺服器回傳的實際欄位值回填到 mock 資料物件中（例如 auto-generated name）。

---

### `sliceIntoChunks(array, size)`

將陣列切割為指定大小的等分。用於 `insertUniAPI` 的批次新增（每批最多 8 筆，避免 timeout）。

---

## Checkout Helpers（tasks/common/utils.test.ts）

位於 `tasks/common/utils.test.ts` 中的結帳流程 helpers：

| 函式 | 說明 |
|------|------|
| `generateBatchInsertTestTask(params)` | 根據 BatchInsertParam 陣列，為每筆資料動態產生一個 `it()` 測試項目 |
| `enterEntryTask(entry, autoSetupSaler?)` | 導航到指定介面（item / salon / workbench） |
| `backToMainMenuTask(entry, member)` | 回到主選單並驗證資料已清空 |
| `addItemToList(items)` | 在掃碼欄位輸入商品名稱加入購物車 |
| `enterCheckout(page, opts)` | 進入結帳畫面 |
| `selectPosMember(member)` | 選取會員 |
| `setPayInfo(info)` | 設定付款資訊（現金/信用卡等） |
| `setupSaler(page, saler)` | 設定銷售人員 |
| `clearAll()` | 清空購物車 |
| `finishCheckout(page)` | 完成結帳流程 |
| `setEGUIOrUniNo(page, value, opts)` | 設定電子發票載具或統一編號 |
| `getTotalBeforeCheckout(page)` | 取得結帳前金額 |
| `findItemTableRows(page)` | 查找商品列表中的行 |
| `waitItemListIdle(page)` | 等待商品列表載入完成 |

**`generateBatchInsertTestTask` 特別說明：**

```typescript
export function generateBatchInsertTestTask(params: BatchInsertParam[], ignoreError?) {
  params.forEach(({ tableName, displayName, records, options }) => {
    it(`新增 ${displayName} 測試用資料`, async () => {
      // 每批最多 8 筆，避免 timeout
      const insertedIds = await Promise.all(
        sliceIntoChunks(records, 8).map(chunk => insertUniAPI(...))
      );
      if (afterInsert) await afterInsert(insertedIds.flat());
    }, 2 * 60 * 1000);  // timeout 2 分鐘
  });
}
```

此函式會**在 Jest 註冊階段同步產生 `it()` 項目**，因此必須在 `describe()` block 的頂層呼叫，不能在 `beforeAll` 或其他 async context 中使用。
