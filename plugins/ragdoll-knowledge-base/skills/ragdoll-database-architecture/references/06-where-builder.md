# WHERE 條件建構器

程式碼位置：`electron/main/database/drizzle-where-builder.ts`

---

## 概述

將 MongoDB 風格的 GraphQL filter 物件轉換為 Drizzle ORM 的 WHERE 條件。
取代舊版的 `graphqlFiltersToSQL`（raw SQL 字串拼接），提供型別安全的條件建構。

---

## 三種輸出格式

針對不同的 Drizzle query API，提供三種輸出格式：

| 函式 | 回傳型別 | 適用場景 |
|------|---------|---------|
| `graphqlFiltersToDrizzle()` | `SQL` | 直接 SQL expression，用於 `select().from().where()` |
| `buildDrizzleWhere()` | `{ RAW: callback }` | Relational query 格式，用於 `db.query.xxx.findMany()` |
| `buildDrizzleWhereSql()` | `SQL` | 簡化入口，自動從 TABLE_MAP 查找 table |

### 使用場景對應

```typescript
// 1. graphqlFiltersToDrizzle — 需要 table 物件 + tableName
const where = graphqlFiltersToDrizzle(table, tableName, filters);
db.select().from(table).where(where);

// 2. buildDrizzleWhere — Relational query（list / findOne 使用）
const where = buildDrizzleWhere(tableName, filters);
db.query.posPromotion.findMany({ where });

// 3. buildDrizzleWhereSql — 自動查找 table（count 使用）
const where = buildDrizzleWhereSql(tableName, filters);
db.select({ count: sql`count(*)` }).from(table).where(where);
```

---

## MongoDB 風格運算子

| GraphQL 運算子 | Drizzle API | SQL 等價 | 說明 |
|---------------|------------|---------|------|
| `$in` | `inArray()` | `IN (...)` | 值在列表中 |
| `$nin` | `notInArray()` | `NOT IN (...)` | 值不在列表中 |
| `$gte` | `gte()` | `>=` | 大於等於 |
| `$gt` | `gt()` | `>` | 大於 |
| `$lte` | `lte()` | `<=` | 小於等於 |
| `$lt` | `lt()` | `<` | 小於 |
| `$like` | `like()` | `LIKE '%value%'` | 模糊匹配（自動包裹 `%`） |

### Date 處理

`$gte` / `$gt` / `$lte` / `$lt` 如果值是 `Date` 物件，自動轉為 `date.getTime()`（Unix timestamp ms）。

---

## Filter 結構

```typescript
type GraphQLFilter<T> = {
    body?: Record<string, FieldCondition<FIELD_VALUE_TYPE>>;
    lines?: Record<string, Record<string, FieldCondition<FIELD_VALUE_TYPE>>>;
};

type FieldCondition<T> = {
    $in?: T[];
    $nin?: T[];
    $gte?: T;
    $gt?: T;
    $lte?: T;
    $lt?: T;
    $like?: string[];
};
```

### 條件組合規則

- **同一 filter 內**：不同欄位之間用 **AND** 連接
- **同一欄位內**：多個條件用 **OR** 連接
- **多個 filter 之間**：用 **OR** 連接

```typescript
// 範例：查找名稱包含 "貓" 或 "狗"，且價格大於 100 的商品
const filters = [{
    body: {
        displayName: { $like: ['貓', '狗'] },   // displayName LIKE '%貓%' OR LIKE '%狗%'
        price: { $gt: 100 },                     // AND price > 100
    }
}];
```

---

## Lines 條件 — EXISTS 子查詢

當 filter 包含 `lines` 條件時，使用 `EXISTS` 子查詢來匹配：

```typescript
const filters = [{
    body: { displayName: { $like: ['促銷'] } },
    lines: {
        filters: {
            fieldName: { $in: ['itemBrandName'] },
            fieldValue: { $in: ['BRAND-001'] },
        }
    }
}];
```

產生的 SQL：

```sql
SELECT * FROM pos_promotion
WHERE display_name LIKE '%促銷%'
  AND EXISTS (
    SELECT 1 FROM pos_promotion_lines_filters
    WHERE pos_promotion_lines_filters.pos_promotion_name = pos_promotion.name
      AND field_name IN ('itemBrandName')
      AND field_value IN ('BRAND-001')
  )
```

### $in 包含 null 的特殊處理

當 `$in` 陣列包含 `null` 時，表示要**同時包含沒有子表資料的記錄**：

```typescript
const filters = [{
    lines: {
        filters: {
            fieldName: { $in: [null, 'itemBrandName'] }
        }
    }
}];
```

產生的 SQL：

```sql
-- 有匹配的子表資料 OR 完全沒有子表資料
EXISTS (
    SELECT 1 FROM pos_promotion_lines_filters
    WHERE ... AND field_name IN ('itemBrandName')
)
OR NOT EXISTS (
    SELECT 1 FROM pos_promotion_lines_filters
    WHERE pos_promotion_lines_filters.pos_promotion_name = pos_promotion.name
)
```

處理步驟：
1. 從 `$in` 陣列中移除 `null`
2. 用剩餘值建立 `EXISTS` 子查詢
3. 額外建立 `NOT EXISTS` 子查詢（匹配空子表）
4. 兩者用 `OR` 連接

---

## buildDrizzleWhere — RAW Callback 格式

Drizzle v2 relational query 需要 RAW callback 格式的 WHERE 條件：

```typescript
function buildDrizzleWhere(tableName, filters) {
    return {
        RAW: (cols, ops) => {
            // cols: { name: Column, displayName: Column, ... }
            // ops: { eq, inArray, gte, or, and, ... }
            return ops.and(
                ops.inArray(cols.name, ['PROMO-001', 'PROMO-002']),
                ops.gte(cols.startDate, 1700000000000)
            );
        }
    };
}
```

### 為什麼需要 RAW callback

Drizzle relational query 會將表 alias 為 `d0`、`d1` 等。
直接使用 `eq(table.name, value)` 會指向原始表名，不是 alias。
RAW callback 的 `cols` 參數已經是正確的 aliased column，確保查詢正確。

### 限制

目前 `buildDrizzleWhere` **僅支援 body 條件**。
Lines 條件（EXISTS 子查詢）應透過 relational query 的 `with` 子句在應用層過濾。

---

## WhereFilter 型別

提供型別安全的 filter 建構：

```typescript
type WhereFilter<
    R extends DatabaseRecord,
    B = ExtractDatabaseRecordBody<R>,
    L = ExtractDatabaseRecordLines<R>,
> = {
    body?: { [FieldName in keyof B]?: FieldCondition<NonNullable<B[FieldName]>> };
    lines?: {
        [LineName in keyof L]?: {
            [FieldName in keyof L[LineName]]?: FieldCondition<NonNullable<NonNullable<L[LineName]>[FieldName]>>;
        };
    };
};
```

使用範例：

```typescript
const filter: WhereFilter<PosPromotionLocalRecord> = {
    body: {
        name: { $in: ['PROMO-001'] },      // ✅ 型別安全
        startDate: { $gte: Date.now() },    // ✅ 型別安全
        // nonExistentField: { $in: [] },   // ❌ 編譯錯誤
    },
    lines: {
        filters: {
            fieldName: { $in: ['itemBrandName'] },  // ✅ 型別安全
        }
    }
};
```
