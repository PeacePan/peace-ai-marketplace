# VM Sandbox 隔離執行機制

## 為什麼需要 VM

NorwegianForest 的腳本**不是**普通的 Node.js 模組。它們在正式環境中被 webpack 編譯後載入 vm2 的 `NodeVM` 沙盒執行。沙盒提供：

- **隔離性**：腳本無法存取 Node.js 主程序的全域物件
- **受控的 I/O**：所有資料庫操作透過 `ctx.query` 介面，經 MessagePort 傳回主執行緒處理
- **凍結的共用函式庫**：`dayjs`、`Utils` 等 libs 會被 `vm.freeze()` 注入，腳本無法修改

測試時必須走相同的 Sandbox 路徑（`MySandbox.execute`），**只 mock 資料庫查詢（QueryProcessor）**，讓業務邏輯在真實的沙盒環境中執行。

## 為什麼不能直接 import 腳本來測試

你可能會想：「直接 `import policy from '...'` 然後呼叫它不就好了？」

這**不行**，原因如下：

1. **腳本使用 `module.exports`**：webpack 編譯後的產物是 CommonJS 格式，在 VM 中透過 `vm.run()` 載入
2. **`ctx` 物件由 Sandbox 注入**：腳本的第二個參數 `ctx` 包含 `query`、`user`、`env` 等，這些在 VM 外不存在
3. **共用函式庫透過 `vm.freeze()` 注入**：`dayjs`、`Utils` 等是以全域凍結物件的方式提供，不是 import
4. **覆蓋率追蹤**：NYC 的 `instrument` 需要在腳本載入前儀表化，Sandbox 負責將 `__coverage__` 傳回主程序

唯一的例外是 **cron 腳本中 export 的純函式**（不依賴 `ctx`），可以直接 import 測試。

## 執行架構

```
┌─────────────────────────────────────────────────┐
│  主執行緒 (Mocha Test Runner)                     │
│                                                   │
│  1. generateExecuteFunc() 讀取編譯後 JS           │
│  2. nyc instrument 儀表化                         │
│  3. MySandbox.execute() 發送至 Worker              │
│                                                   │
│  ┌─ MySandbox.queryProcessor ──────────────────┐  │
│  │  攔截所有 ctx.query.* 呼叫                    │  │
│  │  根據 query.method + query.table 回傳 mock   │  │
│  └──────────────────────────────────────────────┘  │
│                    ▲                                │
│                    │ MessagePort (BSON)             │
│                    ▼                                │
│  ┌─ Worker Thread (vm2 NodeVM) ────────────────┐  │
│  │  載入凍結的 libs (dayjs, Utils, ...)         │  │
│  │  執行儀表化後的腳本                           │  │
│  │  ctx.query.* → 透過 MessagePort 傳至主執行緒  │  │
│  │  回傳執行結果 + __coverage__                  │  │
│  └──────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

## Worker 執行緒細節

JavaCat 的 MySandbox 使用 **Worker Threads** 池（預設 6 個 worker），每個 worker 內建一個 vm2 NodeVM 實例：

```typescript
// JavaCat 內部實作（簡化）
const vm = new NodeVM({
  console: 'inherit',
  sandbox: { /* __coverage__ 等全域物件 */ },
  env: input.sandboxEnv,
  require: {
    builtin: input.sandboxBuiltins,
    external: input.sandboxExternals,
  }
});

// 凍結共用函式庫
libs?.forEach((lib) => {
  const sandboxedLib = vm.run(lib.script, 'lib/' + lib.name);
  vm.freeze(sandboxedLib, lib.name);
});

// 執行腳本
const sandboxedScript = vm.run(script, name);
const result = await sandboxedScript(...arguments);
```

腳本與主程序之間的通訊使用 **BSON 序列化**，效能優於 JSON 但有型別限制（例如不支援 `undefined`，會被轉為 `null`）。

## 腳本類型與 VM 的關係

| 腳本類型 | 函式簽名 | VM 中的行為 |
|---------|---------|------------|
| **PolicyScript** | `(record, ctx, oldRecord?) → Promise<string \| null>` | 驗證記錄，回傳錯誤訊息或 null |
| **BeforeInsertScript** | `(record, ctx) → Promise<R>` | 轉換記錄後回傳 |
| **BeforeUpdateScript** | `(update, original, ctx) → Promise<data \| string>` | 轉換更新資料或回傳錯誤 |
| **FunctionScript** | `(record, argument, ctx) → Promise<P>` | 執行自訂邏輯，回傳結果 |
| **CallFunctionScript** | `(null, argument, ctx) → Promise<R>` | 獨立函式，不綁定記錄 |
| **ApprovalHookScript** | `(record, ctx) → Promise<string>` | 判斷簽核鏈名稱 |
| **BatchPolicyScript** | `({records, ctx}) → Promise<string \| null>` | 批次驗證 |
| **TodoJobScript** | 多階段執行，回傳 `{status, buffer}` | 長時間運行的批次工作 |
| **CronScript** | 排程觸發 | 定時執行邏輯 |
