# 雙資料庫架構

程式碼位置：
- 初始化入口：`electron/main/database/index.ts`
- Mutable client：`electron/main/database/mutable-client.ts`
- Readonly client：`electron/main/database/readonly-client.ts`

---

## 設計理念

Ragdoll 使用兩個獨立的 SQLite 資料庫檔案，依據**資料來源**和**可恢復性**分離：

| 屬性 | Mutable DB | Readonly DB |
|------|-----------|-------------|
| 檔案名稱 | `offline-mutable.db` | `offline-readonly.db` |
| 資料來源 | 本地結帳流程產生 | 遠端主檔（JavaCat GraphQL + S3） |
| 可否刪除重建 | **不可** — 交易資料不可丟失 | **可以** — 資料隨時可從遠端重新拉取 |
| Migration 失敗策略 | Schema 停舊版，不刪 DB | 刪掉重建 + 全量同步 |
| 表格數量 | 2 張主表 + 3 張子表 | 19 張主表 + 42 張子表 |

### 故障隔離優勢

- Readonly DB 損壞時可直接刪除重建 + 全量同步，**不影響交易資料**
- Mutable DB 的交易資料（offline_sale、issue_invoice）永遠獨立保護
- 兩個 DB 各自的 migration 互不影響

### 新表格放哪

- 遠端資料（GraphQL / S3 同步）→ **Readonly DB**（`schema/readonly/`）
- 本地產生的資料（結帳、發票）→ **Mutable DB**（`schema/mutable/`）

---

## Mutable DB 表格清單

| 主表 | 子表 | 說明 |
|------|------|------|
| `offline_sale` | `offline_sale_lines_items` | 離線銷售單 — 商品明細 |
| | `offline_sale_lines_promotions` | 離線銷售單 — 促銷記錄 |
| | `offline_sale_lines_payments` | 離線銷售單 — 付款記錄 |
| `issue_invoice` | （無子表） | 發票開立紀錄 |

## Readonly DB 表格清單

| 主表 | 子表數 | 說明 |
|------|--------|------|
| `item` | 1（bar_codes） | 商品主檔 |
| `item_animal` | 0 | 動物分類 |
| `item_brand` | 0 | 品牌 |
| `item_category` | 0 | 主分類 |
| `item_subcategory` | 0 | 次分類 |
| `item_small_category` | 0 | 小分類 |
| `pos_store` | 0 | 門市 |
| `pos_config` | 0 | 機台配置 |
| `pos_store_group` | 1（stores） | 門市群組 |
| `pos_promotion` | 7 | 商品促銷 |
| `pos_promotion_snapshot` | 7 | 商品促銷快照 |
| `pos_coupon` | 5 | 折扣碼 |
| `pos_coupon_snapshot` | 5 | 折扣碼快照 |
| `pos_order_discount` | 4 | 整單折扣 |
| `pos_order_discount_snapshot` | 4 | 整單折扣快照 |
| `pos_freebie` | 6 | 加購贈品活動 |
| `pos_item_collection` | 1（items） | 料件活動集合 |
| `pos_egui_no` | 1（number_interval） | 發票配號 |
| `pos_printable_coupon` | 3 | 小白單優惠券 |
| `user` | 0 | 使用者 |

> **注意**：`item` 和 `pos_egui_no` 在 table-registry 中標記為 `'all'`，
> 因為它們雖然來源是遠端同步，但本地也會寫入（促銷價/員工價計算、發票號碼位移量更新）。

---

## 初始化流程

`initialDatabase()` 是應用啟動時的資料庫初始化入口，包含三個步驟：

```
[Step 1] initMutableDb()
         ├─ 開啟 offline-mutable.db 連線
         ├─ 設定 PRAGMA (WAL / synchronous=FULL / foreign_keys=ON)
         ├─ ensureDrizzleMigrationTable() — 舊版 DB 相容
         ├─ 建立 Drizzle instance (schema + relations + casing)
         └─ migrate() — 執行未完成的 migration
              ↓
[Step 2] findLatestInitDatabase()
         ├─ 掃描 resources/data/ 目錄
         ├─ 解析 offline-readonly-vN-YYYYMMDDhhmm.gz 檔名
         ├─ 按版本號（desc）→ 時間戳（desc）排序
         ├─ 比對本地 INIT_READONLY_DATABASE_TIMESTAMP
         └─ 若較新：解壓 .gz → tmpdir → copyFileSync 覆蓋 DB
              ↓
[Step 3] initReadonlyDb()
         ├─ 開啟 offline-readonly.db 連線
         ├─ 設定 PRAGMA (WAL / cache_size=-64000 / temp_store=MEMORY)
         ├─ ensureDrizzleMigrationTable() — 舊版 DB 相容
         ├─ 建立 Drizzle instance
         └─ migrate() — 補上 .gz 版本之後的 migration
```

---

## .gz 離線資料庫版本管理

Readonly DB 支援透過預先建好的 `.gz` 壓縮檔初始化，避免首次啟動時的全量同步等待。

### 檔名格式

```
offline-readonly-v{版本號}-{YYYYMMDDHHmm}.gz
```

- **版本號**（integer）：schema 版本，優先排序依據
- **時間戳**（12 位數字）：資料快照時間，同版本內的排序依據

### 版本比對邏輯

```typescript
// findLatestInitDatabase() 排序規則
dbFiles.sort((a, b) => {
    if (a.version !== b.version) return b.version - a.version; // 版本大者優先
    return b.timestamp - a.timestamp;                           // 同版本取最新時間戳
});
```

### 更新條件

當以下任一條件成立時，會解壓覆蓋現有 DB：
1. DB 檔案不存在（首次安裝）
2. `.gz` 時間戳大於 electron-store 中的 `INIT_READONLY_DATABASE_TIMESTAMP`

### 更新後的 store 寫入

```typescript
store.set('LAST_OFFLINE_ITEM_SYNC_AT', latestInit.timestamp);
store.set('LAST_DATA_CHECK_AT', latestInit.timestamp);
store.set('INIT_READONLY_DATABASE_TIMESTAMP', latestInit.timestamp);
```

---

## PRAGMA 配置

### Mutable DB

```sql
PRAGMA journal_mode = WAL;      -- Write-Ahead Logging，提升並發讀寫效能
PRAGMA synchronous = FULL;      -- 最安全的同步模式，確保交易資料不丟失
PRAGMA foreign_keys = ON;       -- 啟用外鍵約束
```

### Readonly DB

```sql
PRAGMA journal_mode = WAL;      -- Write-Ahead Logging
PRAGMA cache_size = -64000;     -- 64MB 快取（負值 = KiB），加速大量主檔查詢
PRAGMA temp_store = MEMORY;     -- 暫存表使用記憶體，加速排序和聚合
```

> **注意**：Readonly DB 不設 `synchronous = FULL`（效能優先），也不設 `foreign_keys = ON`
> （同步時需要關閉外鍵約束以支援先刪後寫的替換模式）。

---

## 連線生命週期

### 主線程（Singleton 模式）

```typescript
// 初始化（應用啟動時）
initMutableDb();   // → 建立 sqlite + db singleton
initReadonlyDb();  // → 建立 sqlite + db singleton

// 使用（IPC handler 等場景）
getMutableDb();    // → 回傳 Drizzle instance
getReadonlyDb();   // → 回傳 Drizzle instance
getMutableSqlite();  // → 回傳 node:sqlite instance（用於 PRAGMA / raw SQL）
getReadonlySqlite(); // → 回傳 node:sqlite instance

// 關閉（應用關閉時）
closeDatabase();   // → closeMutableDb() + closeReadonlyDb()
```

### Worker 線程（獨立連線）

Worker 線程**不能**跨 thread 共享主線程的 `DatabaseSync` 實例，必須建立獨立連線：

```typescript
import { DatabaseSync } from 'node:sqlite';
import { createReadonlyDrizzle } from '../database/readonly-client';

// Worker 內建立獨立連線
const sqlite = new DatabaseSync(readonlyDbPath);
const db = createReadonlyDrizzle(sqlite);

// 使用完畢必須關閉
sqlite.close();
```

- 不需要跑 `migrate()`（主線程啟動時已完成）
- 使用 `createReadonlyDrizzle()` 確保 schema / relations / casing 配置一致
