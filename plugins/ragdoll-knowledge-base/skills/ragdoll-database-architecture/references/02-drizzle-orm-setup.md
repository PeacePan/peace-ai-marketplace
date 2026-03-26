# Drizzle ORM 設定

程式碼位置：
- Mutable client：`electron/main/database/mutable-client.ts`
- Readonly client：`electron/main/database/readonly-client.ts`
- Mutable config：`electron/main/database/drizzle.config.mutable.ts`
- Readonly config：`electron/main/database/drizzle.config.readonly.ts`

---

## 技術棧

| 元件 | 版本 | 說明 |
|------|------|------|
| Drizzle ORM | v1 | ORM 層，提供型別安全的 query builder |
| node:sqlite | Node.js 內建 | SQLite 驅動（同步 API，非 async） |
| drizzle-kit | — | CLI 工具，負責 migration 產生與管理 |

> **為什麼選擇 node:sqlite**：取代了原本的 better-sqlite3，因為 node:sqlite 是 Node.js 內建模組，
> 不需要 native binding 編譯，減少 Electron 打包問題。
> 詳見架構決策文件：`.claude/skills/ragdoll-project-knowledge/references/decisions/drizzle-v1-node-sqlite.md`

---

## Drizzle Instance 建立

### Mutable DB

```typescript
// mutable-client.ts
import * as mutableSchema from './schema/mutable/offline-sale';
import * as issueInvoiceSchema from './schema/mutable/issue-invoice';
import { relations as mutableRelations } from './schema/mutable/relations';

const schema = { ...mutableSchema, ...issueInvoiceSchema };

db = drizzle({
    client: sqlite,           // node:sqlite DatabaseSync instance
    schema,                   // 所有 mutable 表的 schema 定義
    relations: mutableRelations, // Drizzle v1 defineRelations 結果
    casing: 'snake_case',     // camelCase → snake_case 自動轉換
});
```

### Readonly DB

```typescript
// readonly-client.ts
import * as readonlySchema from './schema/readonly';
import { relations as readonlyRelations } from './schema/readonly/relations';

// 公開的工廠函式，也供 Worker 使用
export function createReadonlyDrizzle(client: DatabaseSync) {
    return drizzle({
        client,
        schema: readonlySchema,
        relations: readonlyRelations,
        casing: 'snake_case',
    });
}
```

---

## Casing 規則

Drizzle client 設定 `casing: 'snake_case'`，自動處理 TypeScript ↔ SQL 的命名轉換：

| 層級 | 命名風格 | 範例 |
|------|---------|------|
| TypeScript schema 定義 | **camelCase** | `displayName` |
| SQL 資料庫欄位 | **snake_case**（自動） | `display_name` |
| TypeScript 查詢結果 | **camelCase**（自動） | `result.displayName` |

### 重要：不需要手動轉換

因為 Drizzle 內建的 casing 設定已經自動處理，所以：
- **不需要** `deepKeysToSnake()` 轉換寫入資料
- **不需要** `deepKeysToCamel()` 轉換查詢結果
- Schema 中直接用 `camelCase` 定義欄位名即可

```typescript
// ✅ 正確：直接用 camelCase
export const posPromotion = sqliteTable('pos_promotion', {
    displayName: text().notNull(),  // DB 自動存為 display_name
});

// ❌ 錯誤：不需要手動指定 SQL 欄位名
export const posPromotion = sqliteTable('pos_promotion', {
    displayName: text('display_name').notNull(),  // 多此一舉
});
```

---

## Drizzle Kit 配置

兩個 DB 各有獨立的 drizzle-kit 配置檔：

### drizzle.config.mutable.ts

```typescript
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
    dialect: 'sqlite',
    schema: './electron/main/database/schema/mutable/*.ts',
    out: './electron/main/database/migrations/mutable',
    casing: 'snake_case',
});
```

### drizzle.config.readonly.ts

```typescript
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
    dialect: 'sqlite',
    schema: './electron/main/database/schema/readonly/*.ts',
    out: './electron/main/database/migrations/readonly',
    casing: 'snake_case',
});
```

### 配置要點

- `schema`：使用 glob 指向對應 DB 的 schema 目錄
- `out`：migration 檔案輸出到各自的 `migrations/` 子目錄
- `casing`：與 Drizzle client 一致，使用 `snake_case`
- `dialect`：固定為 `sqlite`

---

## Relations 定義

Drizzle v1 使用 `defineRelations()` 集中定義表之間的關聯，分別定義在：
- `schema/mutable/relations.ts`
- `schema/readonly/relations.ts`

### 範例：Mutable Relations

```typescript
import { defineRelations } from 'drizzle-orm';
import * as schema from './offline-sale';
import * as issueInvoiceSchema from './issue-invoice';

export const relations = defineRelations(
    { ...schema, ...issueInvoiceSchema },
    (r) => ({
        offlineSale: {
            items: r.many.offlineSaleLinesItems(),
            promotions: r.many.offlineSaleLinesPromotions(),
            payments: r.many.offlineSaleLinesPayments(),
        },
        offlineSaleLinesItems: {
            offlineSale: r.one.offlineSale({
                from: r.offlineSaleLinesItems.offlineSaleName,
                to: r.offlineSale.name,
            }),
        },
        // ... 其他子表的反向關聯
    })
);
```

### Relations 的用途

1. **Relational Query**：`db.query.posPromotion.findMany({ with: { filters: true, items: true } })`
   → 自動 JOIN 子表
2. **TABLE_MAP 自動推導**：`buildTableMap()` 從 relations 配置中提取主表 ↔ 子表映射
3. **型別推導**：Drizzle 從 relations 推導 `with` 子句的合法選項

---

## Schema 檔案結構

```
database/schema/
├── index.ts              ← 匯出所有 schema（readonly re-export）
├── types.ts              ← 型別系統（DatabaseRecord, ToInsert, Body/Lines 型別）
├── common.ts             ← commonColumns() 共用欄位
├── table-registry.ts     ← 表格分類（readonly / mutable / all）
├── table-map.ts          ← TABLE_MAP 自動推導（主表 → 子表映射）
│
├── mutable/
│   ├── offline-sale.ts   ← offline_sale + 3 張子表
│   ├── issue-invoice.ts  ← issue_invoice
│   └── relations.ts      ← mutable 表的 defineRelations
│
└── readonly/
    ├── index.ts           ← re-export 所有 readonly schema
    ├── item.ts            ← item + item_lines_bar_codes
    ├── item-animal.ts     ← item_animal
    ├── item-brand.ts      ← item_brand
    ├── item-category.ts   ← item_category
    ├── item-subcategory.ts ← item_subcategory
    ├── item-small-category.ts ← item_small_category
    ├── pos-store.ts       ← pos_store
    ├── pos-config.ts      ← pos_config
    ├── pos-store-group.ts ← pos_store_group + lines
    ├── pos-promotion.ts   ← pos_promotion + 7 張子表
    ├── pos-promotion-snapshot.ts ← pos_promotion_snapshot + 7 張子表
    ├── pos-coupon.ts      ← pos_coupon + 5 張子表
    ├── pos-coupon-snapshot.ts ← pos_coupon_snapshot + 5 張子表
    ├── pos-order-discount.ts ← pos_order_discount + 4 張子表
    ├── pos-order-discount-snapshot.ts ← pos_order_discount_snapshot + 4 張子表
    ├── pos-freebie.ts     ← pos_freebie + 6 張子表
    ├── pos-item-collection.ts ← pos_item_collection + lines
    ├── pos-egui-no.ts     ← pos_egui_no + lines
    ├── pos-printable-coupon.ts ← pos_printable_coupon + 3 張子表
    ├── user.ts            ← user
    └── relations.ts       ← readonly 表的 defineRelations
```
