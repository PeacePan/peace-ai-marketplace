# Schema 定義與型別系統

程式碼位置：
- 共用欄位：`electron/main/database/schema/common.ts`
- 型別系統：`electron/main/database/schema/types.ts`
- 表格映射：`electron/main/database/schema/table-map.ts`
- 表格分類：`electron/main/database/schema/table-registry.ts`

---

## Schema 定義慣例

### 主表定義

```typescript
/** 商品促銷檔 */
export const posPromotion = sqliteTable('pos_promotion', {
    /** 促銷名稱（主鍵） */
    name: text().primaryKey().notNull(),
    /** 顯示名稱 */
    displayName: text().notNull(),
    /** 開始日期 (Unix timestamp ms) */
    startDate: int(),
    ...commonColumns('timestamps'),  // ← createdAt + updatedAt
});
```

### 子表定義

子表命名規則：`{主表}_lines_{子表名}`

```typescript
/** 促銷篩選條件子表 */
export const posPromotionLinesFilters = sqliteTable('pos_promotion_lines_filters', {
    ...commonColumns('lineId', 'timestamps'),  // ← rowId(PK) + createdAt + updatedAt
    /** 所屬促銷（外鍵） */
    posPromotionName: text().notNull().references(() => posPromotion.name),
    /** 篩選欄位名 */
    fieldName: text().notNull(),
    /** 篩選值 */
    fieldValue: text().notNull(),
});
```

### 命名規範要點

| 項目 | 規範 |
|------|------|
| 主鍵 | 一律使用 `name: text().primaryKey().notNull()` |
| 子表命名 | `{主表SQL名}_lines_{子表名}` — `_lines_` 是 TABLE_MAP 自動識別的依據 |
| 外鍵命名 | `{主表camelCase}Name`（如 `posPromotionName`） |
| 外鍵定義 | `.references(() => mainTable.name)` |
| 每個欄位 | **必須**加 JSDoc 註解 |

---

## commonColumns — 共用欄位

`commonColumns()` 提供兩組 preset，透過解構合併到 `sqliteTable` 定義中：

### timestamps preset

```typescript
{
    /** 建立時間 (Unix timestamp ms) */
    createdAt: int().$defaultFn(() => Date.now()),
    /** 更新時間 (Unix timestamp ms) */
    updatedAt: int().$onUpdateFn(() => Date.now()),
}
```

- `$defaultFn`：INSERT 時自動填入 `Date.now()`
- `$onUpdateFn`：UPDATE 時自動更新為 `Date.now()`
- 在 `InferInsertModel` 中自動變成 **optional**（寫入時不需提供）

### lineId preset

```typescript
{
    /** 子表行 ID（自動生成 UUID） */
    rowId: text().primaryKey().$defaultFn(() => crypto.randomUUID()),
}
```

- 子表的主鍵，自動生成 UUID
- 在 `InferInsertModel` 中自動 **optional**

### 使用方式

```typescript
// 主表 — 只需 timestamps
sqliteTable('item', {
    name: text().primaryKey().notNull(),
    ...commonColumns('timestamps'),
});

// 子表 — 需要 lineId + timestamps
sqliteTable('item_lines_bar_codes', {
    ...commonColumns('lineId', 'timestamps'),
    itemName: text().notNull().references(() => item.name),
});
```

---

## 型別系統

### DatabaseRecord — 統一資料格式

整個應用使用 `{ body, lines }` 格式在各層之間傳遞資料：

```typescript
type DatabaseRecord<B, L> = {
    /** 主表表頭資料 */
    body: B;
    /** 子表表身資料陣列 */
    lines?: Partial<{ [LN in keyof L]: L[LN][] }>;
};
```

### Body 型別（查詢結果）

從 Drizzle schema 自動推導，代表完整的查詢結果行：

```typescript
type ItemBody = InferSelectModel<typeof item>;
// → { name: string; displayName: string; createdAt: number | null; ... }
```

### Insert Body 型別（寫入用）

系統欄位因為有 `$defaultFn` 所以自動 optional：

```typescript
type ItemInsertBody = InferInsertModel<typeof item>;
// → { name: string; displayName: string; createdAt?: number | null; ... }
```

### Lines 型別

手動定義子表的型別映射：

```typescript
type PosPromotionLines = {
    filters: InferSelectModel<typeof posPromotionLinesFilters>;
    items: InferSelectModel<typeof posPromotionLinesItems>;
    effects: InferSelectModel<typeof posPromotionLinesEffects>;
    storeGroups: InferSelectModel<typeof posPromotionLinesStoreGroups>;
    weekly: InferSelectModel<typeof posPromotionLinesWeekly>;
    monthly: InferSelectModel<typeof posPromotionLinesMonthly>;
    zones: InferSelectModel<typeof posPromotionLinesZones>;
};
```

### ToInsert — 寫入型別轉換

將查詢型別轉為寫入型別，三個步驟：

```typescript
type ToInsert<T> =
    NullableToOptional<Omit<T, SystemManagedFields>>  // 1. 移除系統欄位 + nullable 變 optional
    & Partial<Pick<T, Extract<SystemManagedFields, keyof T>>>; // 2. 系統欄位變 optional
```

1. **移除系統欄位**（`createdAt` / `updatedAt` / `rowId`）→ 變 optional
2. **nullable 欄位**（`T | null`）→ 變 optional 並接受 `undefined`
3. **notNull 欄位**：維持 required

```typescript
// 轉換前
type ItemBody = { name: string; memo: string | null; createdAt: number | null };
// 轉換後
type ItemInsert = ToInsert<ItemBody>;
// → { name: string; memo?: string | null | undefined; createdAt?: number | null | undefined }
```

### LocalTableRecords — 表名 → 型別映射

```typescript
type LocalTableRecords = {
    item: DatabaseRecord<ItemBody, ItemLines>;
    pos_promotion: DatabaseRecord<PosPromotionBody, PosPromotionLines>;
    offline_sale: DatabaseRecord<OfflineSaleBody, OfflineSaleLines>;
    // ... 所有 21 張主表
};

type LocalTableNames = keyof LocalTableRecords;  // 'item' | 'pos_promotion' | ...
```

### LocalTableInsertRecords — 寫入版映射

```typescript
type LocalTableInsertRecords = {
    [K in keyof LocalTableRecords]: DatabaseRecord<
        ToInsert<Body>,
        { [LN in keyof Lines]: Partial<ToInsert<Lines[LN]>> }
    >;
};
```

---

## TABLE_MAP — 主表 ↔ 子表自動映射

`TABLE_MAP` 在啟動時從 Drizzle 的 `defineRelations` 結果自動建立，取代手動維護的映射。

### 資料結構

```typescript
type TableMapEntry = {
    dbName: string;        // SQL 表名 (snake_case)
    tsName: string;        // TypeScript 變數名 (camelCase)，用於 db.query[tsName]
    table: SQLiteTable;    // Drizzle table 物件引用
    lines: Record<string, { dbName: string; table: SQLiteTable }>;  // 子表映射
};

// TABLE_MAP: Record<LocalTableNames, TableMapEntry>
```

### 建立流程

`buildTableMap()` 遍歷所有 relations 配置：
1. 跳過 SQL 表名包含 `_lines_` 的（這些是子表，不建立獨立 entry）
2. 遍歷每個主表的 relations，找出指向 `_lines_` 表的 relation
3. 建立 `{ dbName → { tsName, table, lines } }` 映射

### toRecord() — 扁平結果轉換

Drizzle relational query 回傳的是扁平物件，`toRecord()` 根據 TABLE_MAP 的 lines 定義自動分離：

```typescript
// Drizzle 回傳的扁平物件
{ name: 'PROMO-001', displayName: '滿千折百', filters: [...], items: [...] }

// toRecord() 轉換後
{
    body: { name: 'PROMO-001', displayName: '滿千折百' },
    lines: { filters: [...], items: [...] }
}
```

### getOwnLineTables() — 取得子表清單

用於 remove 操作的級聯刪除，只刪除自己的 lines 子表：

```typescript
getOwnLineTables('pos_promotion')
// → [{ dbName: 'pos_promotion_lines_filters', table: ... }, ...]
```

---

## Table Registry — 表格分類

每張表在 `table-registry.ts` 中標記為三種類型之一：

| 類型 | 說明 | 範例 |
|------|------|------|
| `readonly` | 純遠端快取，只透過 sync 寫入 | `pos_promotion`、`user` |
| `mutable` | 本地產生的資料 | `offline_sale`、`issue_invoice` |
| `all` | 遠端同步 + 本地也會寫入 | `item`（促銷價）、`pos_egui_no`（發票位移） |

### 路由邏輯

Operations 層根據 table type 自動選擇 DB：

```typescript
function isReadonlyTable(tableName): boolean {
    // 'readonly' 或 'all' → 使用 readonlyDb
    return type === 'readonly' || type === 'all';
}

function isMutableTable(tableName): boolean {
    // 'mutable' 或 'all' → 使用 mutableDb
    return type === 'mutable' || type === 'all';
}
```

---

## 三層型別安全網

### 第一層：編譯時（TypeScript）

`SqlNameToTable` 從 Drizzle schema 自動推導 SQL 表名 → table 型別映射：

```typescript
type SqlNameToTable = {
    'item': typeof item;
    'pos_promotion': typeof posPromotion;
    // ... 自動推導
};
```

搭配 `_localTableNamesCheck` 確認所有 `LocalTableNames` 都有對應的 schema 定義。

### 第二層：啟動時（validateTableMap）

```typescript
function validateTableMap(map): Record<LocalTableNames, TableMapEntry> {
    const missingTables = LOCAL_TABLE_NAMES.filter(name => !mapKeys.has(name));
    if (missingTables.length > 0) {
        throw new Error(`以下表缺少 entry：${missingTables.join(', ')}`);
    }
}
```

### 第三層：執行時（Operations 層）

各操作函式在執行前會查找 `TABLE_MAP[tableName]`，找不到時拋出錯誤。
