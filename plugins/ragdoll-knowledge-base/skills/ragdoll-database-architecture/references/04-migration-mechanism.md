# Migration 機制與流程

程式碼位置：
- Migration 檔案：`electron/main/database/migrations/{mutable,readonly}/`
- 向後相容：`electron/main/database/migrate-compat.ts`
- Drizzle Kit 配置：`electron/main/database/drizzle.config.{mutable,readonly}.ts`

---

## Migration 檔案結構

```
migrations/
├── mutable/
│   └── 20260320112530_nostalgic_captain_america/
│       ├── migration.sql      ← SQL DDL 陳述句
│       └── snapshot.json      ← Schema 完整快照（供 drizzle-kit 比對用）
│
└── readonly/
    └── 20260320122848_huge_harrier/
        ├── migration.sql
        └── snapshot.json
```

### 命名格式

```
{YYYYMMDDHHmmss}_{random_name}/
```

- **時間戳**：migration 產生的時間，14 位數字
- **名稱**：drizzle-kit 自動產生的隨機名稱（無業務含義）
- 每個 migration 是一個**目錄**，包含 `migration.sql` 和 `snapshot.json`

### migration.sql 內容

包含純 SQL DDL 陳述句，以 `--> statement-breakpoint` 分隔：

```sql
CREATE TABLE `offline_sale` (
	`name` text PRIMARY KEY NOT NULL,
	`pos_store_name` text NOT NULL,
	...
);
--> statement-breakpoint
CREATE TABLE `offline_sale_lines_items` (
	...
);
--> statement-breakpoint
CREATE INDEX `idx_offline_paid_at` ON `offline_sale` (`paid_at`);
```

### snapshot.json 內容

Schema 的完整快照，包含所有表、欄位、索引、外鍵的定義。
drizzle-kit 使用它來比對當前 schema 與上次 migration 的差異，產生增量 migration。

---

## npm scripts

| 指令 | 用途 | 說明 |
|------|------|------|
| `npm run db:generate` | 產生 migration | 從 schema 差異產生 SQL migration 檔案 |
| `npm run db:push` | 推送到本地 DB | 直接修改 DB schema，不產生 migration 檔案 |
| `npm run db:check` | 檢查一致性 | 驗證 migration 檔案與 schema 是否同步 |
| `npm run db:studio` | 開啟 Drizzle Studio | 視覺化 DB 管理介面 |
| `npm run db:migrate` | 執行 migration | 呼叫 `initReadonlyDb()` + `initMutableDb()` |
| `npm run db:reset:dev` | 重置開發 DB | 刪除 .db 檔案 → 重建初始 DB |
| `npm run db:rm-table:dev` | 移除表格（開發） | 從開發 DB 移除指定表格 |
| `npm run db:rm-table` | 移除表格（正式） | 從正式 DB 移除指定表格 |

### 指令實際內容

```bash
# db:generate — 兩個 DB 各自產生
drizzle-kit generate --config=electron/main/database/drizzle.config.mutable.ts && \
drizzle-kit generate --config=electron/main/database/drizzle.config.readonly.ts

# db:push — 兩個 DB 各自推送
drizzle-kit push --config=electron/main/database/drizzle.config.mutable.ts && \
drizzle-kit push --config=electron/main/database/drizzle.config.readonly.ts

# db:check — 兩個 DB 各自檢查
drizzle-kit check --config=electron/main/database/drizzle.config.mutable.ts && \
drizzle-kit check --config=electron/main/database/drizzle.config.readonly.ts
```

---

## 日常開發工作流程

### 開發中：使用 db:push

```bash
# 1. 修改 schema/*.ts
# 2. 直接推送到本地 DB（不產生 migration）
npm run db:push
```

`db:push` 會直接修改本地 DB 的 schema，適合快速迭代開發。
**不會**產生 migration 檔案，所以不影響其他開發者。

### Commit 前：產生 migration

```bash
# 1. 刪除本地 DB（確保從零開始驗證）
npm run db:reset:dev

# 2. 從 schema 產生 migration
npm run db:generate

# 3. 驗證 migration 與 schema 一致
npm run db:check
```

**MUST**：commit 前必須 `db:generate`，確保 migration 與 schema 同步。
否則其他開發者拉取後的 DB 會與 schema 不一致。

---

## Migration 執行流程

應用啟動時，`initMutableDb()` 和 `initReadonlyDb()` 各自執行以下流程：

```
[1] ensureDrizzleMigrationTable(sqlite, migrationsFolder)
    │
    ├─ 檢查 __drizzle_migrations 表是否存在
    │  ├─ 存在 → 跳過（正常流程）
    │  └─ 不存在 → 檢查是否有使用者表
    │     ├─ 沒有使用者表 → 跳過（全新 DB，讓 migrate() 建表）
    │     └─ 有使用者表 → 舊版 DB 相容處理
    │        ├─ 建立 __drizzle_migrations 表
    │        ├─ 讀取所有 migration 目錄
    │        ├─ 計算每個 migration.sql 的 SHA-256 hash
    │        └─ 插入記錄，標記為已完成
    │
    ▼
[2] migrate(db, { migrationsFolder })
    │
    ├─ 掃描 migrationsFolder 中的所有 migration 目錄
    ├─ 計算每個 migration.sql 的 SHA-256 hash
    ├─ 與 __drizzle_migrations 表中的記錄比對
    ├─ 只執行未記錄的 migration（增量執行）
    └─ 執行完成後寫入新記錄
```

### __drizzle_migrations 表結構

```sql
CREATE TABLE __drizzle_migrations (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    hash TEXT NOT NULL,        -- migration.sql 的 SHA-256 hash
    created_at NUMERIC         -- 執行時間
);
```

### Hash 比對機制

Drizzle 使用 SHA-256 hash 而非檔名來識別 migration：
- 計算 `migration.sql` 內容的 SHA-256
- 與 `__drizzle_migrations.hash` 欄位比對
- 已存在相同 hash → 跳過
- 未找到 hash → 執行該 migration

---

## 舊版 DB 向後相容

### 問題場景

舊版系統使用 `node:sqlite` + 手寫 SQL 建表，DB 中沒有 `__drizzle_migrations` 表。
升級到 Drizzle ORM 後，`migrate()` 找不到 migration 記錄 → 嘗試重新 `CREATE TABLE` → 失敗（表已存在）。

### 受影響的情況

- 正式環境使用者升版（本地 DB 是舊版建的）
- 開發環境使用舊版 `.gz` 解壓的 readonly DB
- 任何在 Drizzle 遷移之前建立的 DB

### 解法：ensureDrizzleMigrationTable()

```typescript
export function ensureDrizzleMigrationTable(
    sqlite: DatabaseSync,
    migrationsFolder: string
): void {
    // 1. 已有 migration 表 → 正常流程，不做任何事
    const hasMigrationTable = sqlite
        .prepare("SELECT name FROM sqlite_master WHERE type='table' AND name='__drizzle_migrations'")
        .get() != null;
    if (hasMigrationTable) return;

    // 2. 全新 DB（沒有任何使用者表）→ 讓 migrate() 正常建表
    const userTables = sqlite
        .prepare("SELECT name FROM sqlite_master WHERE type='table' AND name NOT LIKE 'sqlite_%' AND name NOT LIKE '__drizzle_%'")
        .all();
    if (userTables.length === 0) return;

    // 3. 舊版 DB：有使用者表但沒有 migration 記錄
    //    建立 __drizzle_migrations 表並標記所有現有 migration 為已完成
    sqlite.exec(`CREATE TABLE IF NOT EXISTS __drizzle_migrations (...)`);

    for (const dirName of migrationDirs) {
        const migrationSql = readFileSync(join(migrationsFolder, dirName, 'migration.sql'), 'utf-8');
        const hash = createHash('sha256').update(migrationSql).digest('hex');
        insertStmt.run(hash, Date.now());
    }
}
```

### 三種情境的處理邏輯

| 情境 | 有 __drizzle_migrations | 有使用者表 | 處理方式 |
|------|------------------------|-----------|---------|
| 正常 DB | ✅ | ✅ | 跳過，讓 migrate() 正常增量執行 |
| 全新 DB | ❌ | ❌ | 跳過，讓 migrate() 從頭建表 |
| 舊版 DB | ❌ | ✅ | 建立記錄表 + 標記已完成 → migrate() 只跑新的 |

---

## Custom Migration 使用時機

`drizzle-kit generate` **無法偵測** rename 操作，會當作「刪舊 + 加新」= **資料丟失**。

### 必須使用 custom migration 的情況

| 操作 | 原因 |
|------|------|
| 改欄位名稱 | `ALTER TABLE RENAME COLUMN` — drizzle-kit 視為刪除 + 新增 |
| 改欄位型別 | SQLite 不支援直接改型別，需重建表 |

### Custom migration 步驟

1. 先用 `db:generate` 產生初始 migration
2. 手動修改 `migration.sql`，改為正確的 `ALTER TABLE RENAME COLUMN` 等語句
3. 同步修改 `snapshot.json`（或重新 generate 讓 snapshot 正確）
4. 執行 `db:check` 確認一致性

---

## Migration 與 .gz 初始 DB 的協作

```
[首次安裝]
.gz 解壓 → 得到已含表結構的 DB
    ↓
ensureDrizzleMigrationTable() → 標記 .gz 中已有的 migration 為完成
    ↓
migrate() → 只執行 .gz 版本之後新增的 migration

[升版]
新版 .gz 覆蓋舊 DB（如果版本更新）
    ↓
ensureDrizzleMigrationTable() → 重新標記
    ↓
migrate() → 補上 .gz 與最新 schema 之間的差異 migration
```

這確保了：
- 使用 `.gz` 初始化的 DB 不會重複執行已包含的 migration
- `.gz` 之後的 schema 變更會透過增量 migration 補上
- 整個流程對使用者透明，不需要手動操作

---

## 交易中的外鍵處理

Migration 和 replace 操作中使用 `PRAGMA defer_foreign_keys = ON`：

```sql
-- 在交易開始時
PRAGMA defer_foreign_keys = ON;

-- 執行 DDL / DML 操作
-- （此時外鍵約束暫時不檢查）

-- 交易 COMMIT 時才檢查外鍵約束
COMMIT;
```

這允許在交易中先刪除被引用的記錄再插入新記錄（先刪後寫的同步模式），
只要 COMMIT 時所有外鍵關係都成立即可。
