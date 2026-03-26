---
name: ragdoll-database-architecture
description: >
  Ragdoll POS 雙 SQLite 資料庫架構完整知識庫。涵蓋 Mutable / Readonly 雙資料庫設計、
  Drizzle ORM v1 + node:sqlite 設定、Schema 定義慣例與型別系統、Migration 產生與執行機制、
  Operations 層五大操作（list / insert / update / remove / replace）、
  以及 GraphQL Filter → Drizzle WHERE 條件轉換器。

  以下情況必須參考此文件再動手：
  - 新增或修改 Drizzle schema（`database/schema/` 目錄）
  - 產生或執行 migration（`db:generate` / `db:push` / `db:migrate`）
  - 修改資料庫初始化流程（`database/index.ts`、client 檔案）
  - 修改 Operations 層的 CRUD 操作邏輯
  - 修改 GraphQL Filter → WHERE 條件轉換邏輯
  - 新增資料表或子表（需了解 TABLE_MAP 自動推導機制）
  - 處理 Worker 線程的資料庫存取
  - 理解 `{ body, lines }` 資料格式的轉換流程
  - 處理 .gz 離線資料庫版本管理
  - 排查 migration 相容性問題（舊版 DB 升級）
---

# Ragdoll 資料庫架構知識庫

程式碼位置：`electron/main/database/`

## 架構總覽

Ragdoll 採用**雙 SQLite 資料庫**架構，透過 **Drizzle ORM v1** + **node:sqlite** 驅動，
所有資料以 `{ body, lines }` 統一格式在 IPC / Next.js / Worker 之間流通。

```
┌─────────────────────────────────────────────────────────┐
│                    Electron Main Process                │
│                                                         │
│  ┌──────────────────┐    ┌───────────────────────────┐  │
│  │  Mutable DB      │    │  Readonly DB              │  │
│  │  offline-mutable │    │  offline-readonly          │  │
│  │  ─────────────── │    │  ─────────────────────── │  │
│  │  offline_sale     │    │  item, pos_promotion ...  │  │
│  │  issue_invoice    │    │  (19 張遠端主檔快取表)    │  │
│  │  ─────────────── │    │  ─────────────────────── │  │
│  │  WAL + FULL sync │    │  WAL + 64MB cache         │  │
│  │  不可刪除重建    │    │  可刪除重建 + 全量同步    │  │
│  └──────────────────┘    └───────────────────────────┘  │
│           │                         │                    │
│           └────────┬────────────────┘                    │
│                    ▼                                     │
│  ┌──────────────────────────────────────────────────┐   │
│  │            Operations Layer (dbOps)               │   │
│  │  list / insert / update / remove / replace        │   │
│  │  ─────────────────────────────────────────────── │   │
│  │  動態表名路由 · { body, lines } 轉換              │   │
│  │  自動 DB 選擇 · 子表級聯刪除 · 批次同步           │   │
│  └──────────────────────────────────────────────────┘   │
│                    │                                     │
│       ┌────────────┼────────────┐                       │
│       ▼            ▼            ▼                       │
│   IPC Handler   Worker      Sync Layer                  │
└─────────────────────────────────────────────────────────┘
```

---

## 各章節參考文件

### 雙資料庫架構
參見 [references/01-dual-database-architecture.md](./references/01-dual-database-architecture.md)：
- Mutable DB vs Readonly DB 的設計理念與故障隔離策略
- `initialDatabase()` 三步初始化流程
- `.gz` 離線資料庫版本管理機制（檔名解析、版本比對、解壓覆蓋）
- PRAGMA 配置差異（WAL、synchronous、cache_size、foreign_keys）
- 連線生命週期管理（singleton 模式、Worker 獨立連線）

### Drizzle ORM 設定
參見 [references/02-drizzle-orm-setup.md](./references/02-drizzle-orm-setup.md)：
- Drizzle v1 + node:sqlite 驅動配置
- `casing: 'snake_case'` 自動轉換規則
- 雙 Drizzle instance 的 schema 與 relations 組裝
- `drizzle.config.mutable.ts` / `drizzle.config.readonly.ts` 配置
- Worker 線程使用 `createReadonlyDrizzle()` 建立獨立實例

### Schema 定義與型別系統
參見 [references/03-schema-definition.md](./references/03-schema-definition.md)：
- 主表與子表命名慣例（`_lines_` 規則）
- `commonColumns()` 共用欄位（timestamps / lineId）與 `$defaultFn` 機制
- 型別系統：`DatabaseRecord<B, L>`、`ToInsert<T>`、`NullableToOptional<T>`
- `TABLE_MAP` 從 defineRelations 自動推導主表 ↔ 子表映射
- `table-registry` 表格分類（readonly / mutable / all）
- `toRecord()` 扁平結果 → `{ body, lines }` 格式轉換
- 三層型別安全網（編譯時 + 執行時驗證）

### Migration 機制與流程
參見 [references/04-migration-mechanism.md](./references/04-migration-mechanism.md)：
- Migration 檔案結構（`{timestamp}_{name}/migration.sql` + `snapshot.json`）
- `drizzle-kit generate` / `push` / `check` 指令用途與差異
- npm scripts 完整清單（`db:generate` / `db:push` / `db:check` / `db:migrate` / `db:reset:dev`）
- `ensureDrizzleMigrationTable()` 舊版 DB 向後相容機制
- Migration 執行流程（hash 比對 → 增量執行）
- Custom migration 使用時機（rename column / change type）
- 日常開發與 commit 前的標準工作流程

### Operations 層
參見 [references/05-operations-layer.md](./references/05-operations-layer.md)：
- 五大操作：list / insert / update / remove / replace
- 雙 API 模式：公開函式（自動 DB）vs Impl 函式（手動 DB）
- `list` / `findOne` / `count`：relational query + WHERE 建構 + 分頁
- `insert`：主表 + 子表交易寫入 + FK 自動填充
- `update`：body 更新 + lines upsert 回退邏輯
- `remove`：跨表 FK 探測 + 顯式級聯刪除（不用 ON DELETE CASCADE 的原因）
- `replace`：兩種同步模式（全量替換 vs 增量 upsert）+ 批次進度回調
- `sanitizeValues()`：node:sqlite 型別轉換（undefined→null、bool→0/1、Date→timestamp）

### WHERE 條件建構器
參見 [references/06-where-builder.md](./references/06-where-builder.md)：
- MongoDB 風格運算子對應表（`$in` / `$nin` / `$gte` / `$gt` / `$lte` / `$lt` / `$like`）
- 三種輸出格式：直接 SQL、RAW callback、Core query
- Lines 子表 EXISTS 子查詢機制
- `$in` 包含 null 時的特殊處理（OR notExists）
- `WhereFilter<R>` 型別定義
