# Operations 層

程式碼位置：`electron/main/database/operations/`

- `index.ts` — 匯出所有操作
- `list.ts` — 查詢操作（list / findOne / count）
- `insert.ts` — 插入操作
- `update.ts` — 更新操作
- `remove.ts` — 刪除操作
- `replace.ts` — 替換操作（資料同步）
- `get-column.ts` — 輔助工具（getColumn / sanitizeValues）

---

## 設計理念

Operations 層是 **Drizzle ORM** 和應用標準資料格式 **`{ body, lines }`** 之間的轉接層。

### 為什麼需要這一層

| 問題 | 解法 |
|------|------|
| IPC/REST 傳入的是**字串表名**，Drizzle 需要**型別化的 table 物件** | 透過 `TABLE_MAP` 動態查找 |
| Drizzle relational query 回傳**扁平物件** | `toRecord()` 自動分離成 `{ body, lines }` |
| 不同表需要用不同的 DB（readonly / mutable） | 根據 `table-registry` 自動選擇 |
| 子表不使用 ON DELETE CASCADE | 顯式級聯刪除 |
| 系統欄位需要自動管理 | Drizzle 的 `$defaultFn` / `$onUpdateFn` |
| 同步需要批次處理 + 進度回調 | `replace` 支援 batchSize + onProgress |

---

## 雙 API 模式

每個操作都提供兩個版本：

| API | 用途 | DB 來源 |
|-----|------|---------|
| `list()` | IPC handler、REST API | 自動從 singleton 取得 |
| `listImpl()` | Worker 線程、sync 層 | 由呼叫方傳入 |

```typescript
// 公開函式 — 自動選擇 DB
export function list(tableName, options) {
    const db = isReadonlyTable(tableName) ? getReadonlyDb() : getMutableDb();
    return listImpl(db, tableName, options);
}

// Impl 函式 — 接受 DB 參數
export function listImpl(db, tableName, options) {
    // 實際查詢邏輯
}
```

### 為什麼需要 Impl

Worker 線程不能共享主線程的 `DatabaseSync` 實例（node:sqlite 限制），
必須自行建立連線並傳入 Drizzle instance。`Impl` 後綴是本專案的命名慣例。

---

## list / findOne / count

### list — 查詢多筆記錄

```typescript
function list(
    tableName: LocalTableNames,
    options?: SQLiteListOptions
): DatabaseRecord[]
```

**SQLiteListOptions**：

| 選項 | 說明 |
|------|------|
| `where` | GraphQL filter 陣列（MongoDB 風格） |
| `orderBy` | 排序欄位與方向 |
| `limit` / `offset` | 分頁 |
| `columns` | 選取特定欄位 |
| `skipLines` | 只查 body，跳過子表 |
| `includeLines` | 選擇性包含特定子表 |

**內部流程**：
1. 從 `TABLE_MAP` 取得 `tsName`（db.query 的 key）
2. 組裝 `where` 條件（透過 `buildDrizzleWhere`）
3. 組裝 `orderBy`（動態欄位名 → `asc()` / `desc()`）
4. 組裝 `with` 子句（子表 include）
5. 使用 `db.query[tsName].findMany()` 執行 relational query
6. 每筆結果透過 `toRecord()` 轉換成 `{ body, lines }` 格式

### findOne — 查詢單筆

```typescript
function findOne(
    tableName: LocalTableNames,
    options?: Omit<SQLiteListOptions, 'limit' | 'offset'>
): DatabaseRecord | undefined
```

內部使用 `db.query[tsName].findFirst()`，回傳單筆或 `undefined`。

### count — 計算筆數

```typescript
function count(
    tableName: LocalTableNames,
    options?: { where?: GraphQLFilter[] }
): number
```

使用 `db.select({ count: sql\`count(*)\` }).from(table).where(...)` 執行。
不使用 relational query，直接用 core query API。

---

## insert — 插入記錄

```typescript
function insert(
    tableName: LocalTableNames,
    inserts: DatabaseRecord[],
    options?: { db?: NodeSQLiteDatabase }
): { lastInsertRowid: number | bigint }
```

**內部流程**（在 transaction 中）：

1. 遍歷每筆 `DatabaseRecord`
2. `sanitizeValues(body)` → 轉換型別
3. `db.insert(mainTable).values(body)` → 插入主表
4. 遍歷 `lines`，對每個子表：
   - 自動填充 FK 欄位（`{主表名}Name = body.name`）
   - `sanitizeValues(lineRow)` → 轉換型別
   - `db.insert(lineTable).values(lineRows)` → 批次插入子表
5. 回傳最後一筆的 `lastInsertRowid`

**限制**：拒絕 readonly 表（`isReadonlyTable` 檢查會在 replace 中處理）

---

## update — 更新記錄

```typescript
function update(
    tableName: LocalTableNames,
    updates: Array<{ key: KeyLocator; record: Partial<DatabaseRecord> }>
): void
```

**內部流程**（在 transaction 中）：

1. **Body 更新**：
   - `sanitizeValues(body)`
   - `db.update(mainTable).set(body).where(eq(pk, keyValue))`
   - `updatedAt` 由 Drizzle `$onUpdateFn` 自動更新

2. **Lines 更新**（對每個子表）：
   - 嘗試 `db.update(lineTable).set(lineRow).where(eq(rowId, lineRow.rowId))`
   - 如果 `rowId` 不存在（新增的 line）→ 回退到 `db.insert(lineTable).values(lineRow)`
   - 自動填充 FK 欄位

**注意**：update 不會自動刪除 lines 中被移除的行。
如果需要完整替換 lines，應使用 remove + insert 或 replace。

---

## remove — 刪除記錄

```typescript
function remove(
    tableName: LocalTableNames,
    removes: KeyLocator[]
): RemoveReturnInfo[]
```

**KeyLocator** 格式：`{ name: 'name', value: 'PROMO-001' }`

**內部流程**（在 transaction 中）：

1. **探測跨表引用**：
   - 遍歷所有表的子表，檢查是否有 FK 指向要刪除的主表
   - 例如：刪除 `pos_store_group` 時，`pos_promotion_lines_store_groups` 中的
     `posStoreGroupName` 會引用它
   - 先刪除這些跨表引用的子表行

2. **刪除自己的 lines**：
   - 透過 `getOwnLineTables()` 取得子表清單
   - 依序刪除每個子表中匹配 FK 的行

3. **刪除主表**：
   - `db.delete(mainTable).where(eq(pk, keyValue))`

### 為什麼不用 ON DELETE CASCADE

```
Readonly 表之間有交叉引用：
  pos_promotion → pos_item_collection（透過 lines）
  pos_coupon → pos_store_group（透過 lines）

同步流程是「先刪全部 → 再插入新資料」。
如果使用 CASCADE，刪除 pos_store_group 時會連帶刪除
pos_coupon 中引用它的 lines — 但這不是我們的意圖。

因此必須顯式控制刪除範圍，只刪自己的 lines。
```

---

## replace — 替換（資料同步）

```typescript
function replace(
    tableName: LocalTableNames,
    records: DatabaseRecord[],
    options?: {
        deleteAll?: boolean;       // 預設 true
        batchSize?: number;        // 預設 500
        onProgress?: (progress: ReplaceProgress) => void;
    }
): void
```

**僅限 readonly 表**（資料同步用途）。

### 兩種模式

| 模式 | `deleteAll` | 用途 | 邏輯 |
|------|------------|------|------|
| 全量替換 | `true`（預設） | S3 檔案同步 | 刪除所有記錄 → 批次插入 |
| 增量 upsert | `false` | GraphQL 增量同步 | INSERT OR REPLACE |

### 全量替換流程

```
[transaction 開始]
    PRAGMA defer_foreign_keys = ON
    ↓
    刪除所有 lines 子表記錄
    刪除所有主表記錄
    ↓
    批次插入新記錄（batchSize = 500）
    ├─ 主表 insert
    ├─ 子表 insert（含 FK 自動填充）
    └─ onProgress 回調
    ↓
[transaction COMMIT]
```

### 增量 upsert 流程

```
[transaction 開始]
    遍歷每筆記錄：
    ├─ db.insert(mainTable).values(body).onConflictDoUpdate()
    └─ 對每個子表：
       ├─ 刪除該主表 name 下的舊 lines
       └─ 插入新 lines
    ↓
[transaction COMMIT]
```

### 進度回調

```typescript
type ReplaceProgress = {
    current: number;   // 目前已處理筆數
    total: number;     // 總筆數
};
```

---

## sanitizeValues — 型別轉換

`node:sqlite` 只接受 `null` / `number` / `string` / `Uint8Array` 作為 SQL 參數。
`sanitizeValues()` 負責轉換不支援的型別：

| JavaScript 型別 | 轉換結果 |
|----------------|---------|
| `undefined` | `null` |
| `null` | `null`（不變） |
| `boolean` | `0` / `1` |
| `NaN` / `Infinity` | `null` |
| `Date` | `date.getTime()`（Unix timestamp ms） |
| `object` / `array` | `JSON.stringify()` |
| `string` / `number` | 不變 |

```typescript
sanitizeValues({
    name: 'ITEM-001',         // → 'ITEM-001'（不變）
    isActive: true,           // → 1
    memo: undefined,          // → null
    tags: ['food', 'pet'],    // → '["food","pet"]'
    createdAt: new Date(),    // → 1711234567890
});
```

---

## getColumn / findColumn — 欄位存取

### getColumn — 必定取得

```typescript
function getColumn(table: SQLiteTable, columnName: string): SQLiteColumn
// 找不到時拋出 Error
```

### findColumn — 探測性取得

```typescript
function findColumn(table: SQLiteTable, columnName: string): SQLiteColumn | undefined
// 找不到時回傳 undefined（用於跨表 FK 探測）
```

這兩個函式封裝了 Drizzle table 物件的動態存取邏輯，
因為動態表名場景無法靜態推導 column 型別，需要透過 `as` 轉型。
