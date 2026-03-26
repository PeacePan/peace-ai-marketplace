# Sandbox VM 架構

腳本運行在隔離的 Node.js Worker Thread VM 中，透過 `MySandbox` 管理。

## Worker Pool 設定

```typescript
// JavaCat/src/lib2/MySandbox/index.ts
const __pool = new StaticPool({
  size: 6,                        // 6 個 worker
  task: __filepath,               // 編譯後的 worker.js
  resourceLimits: {
    maxOldGenerationSizeMb: 256,  // Old generation heap
    maxYoungGenerationSizeMb: 256,// Young generation heap (共 512MB)
    codeRangeSizeMb: 64,
    stackSizeMb: 32,
  },
  shareEnv: true,
});
```

- **並發限制**：最多 6 個腳本同時執行（超出排隊等待）
- **記憶體限制**：每個 worker 最多 512MB，超出會被強制終止
- **執行逾時**：預設 120 秒（`(input.timeout || 120) * 1000`），可透過腳本的 `timeout` 參數調整
- **環境隔離**：每個腳本在獨立的 VM 中執行，腳本之間不共享狀態

## 通訊機制

VM 與 master 之間透過 `MessageChannel` 使用 BSON 序列化雙向通訊：

```
腳本 VM (worker)  ←→  MessageChannel (BSON)  ←→  Master (主程序)
      port2                                          port1
```

- 腳本透過 `ctx.query` 向 master 發送 DB 查詢請求
- Master 執行查詢後透過 channel 回傳結果
- 所有資料均以 BSON 格式序列化（支援 Date、Buffer 等原生型別）

## ctx.query 支援的方法

腳本內透過 `ctx.query` 存取 DocumentDB，對應 `MyQuery` 的方法：

### 查詢類
| 方法 | 說明 |
|-----|------|
| `find` | 查詢多筆（舊版） |
| `findV2` | 查詢多筆（新版，支援 selects） |
| `findAllByNames` | 以 name 陣列批量查詢 |
| `docdbFind` | 直接查詢 DocumentDB |
| `athena` | 查詢 Athena 資料 |

### 寫入類（需要 session）
| 方法 | 說明 |
|-----|------|
| `insert` / `insertV2` | 新增單筆 |
| `batchInsert` / `batchInsertV2` | 批量新增 |
| `update` / `updateV2` | 更新單筆 |
| `batchUpdate` / `batchUpdateV2` | 批量更新 |
| `archive` / `archiveV2` | 封存單筆 |
| `batchArchive` / `batchArchiveV2` | 批量封存 |
| `unarchive` / `unarchiveV2` | 取消封存單筆 |
| `batchUnarchive` / `batchUnarchiveV2` | 批量取消封存 |
| `move` | 移動紀錄 |

### insert vs update 參數結構差異

`insertV2` 與 `updateV2` 的頂層參數結構**不同**，容易混淆：

```typescript
// insertV2：新增資料包在 data 內
await ctx.query.insertV2<BodyType>({
  table: 'tablename',
  data: {
    body: { field1: value1, field2: value2 },
    lines: { items: [...] },  // 選填
  },
});

// updateV2：body 和 key 直接在頂層
await ctx.query.updateV2<BodyType>({
  table: 'tablename',
  key: { name: 'name', value: recordName },
  body: { field1: newValue },
});
```

> **常見錯誤**：把 `insertV2` 的 `data.body` 寫成頂層 `body`（誤用 `updateV2` 的格式），會導致插入欄位全部為空。

### 其他
| 方法 | 說明 |
|-----|------|
| `call` | 呼叫 function |
| `publish` / `publishV2` | 發佈事件 |
| `sendEvent` | 發送事件 |
| `todoJobs` / `todoJobsV2` | 查詢待辦工作 |
| `setTodoJob` | 設定待辦工作 |
| `http.json` | HTTP 請求 |

## 注意事項

- **session 必要性**：寫入類方法（insert/update/archive 等）必須在有 session 的 context 中執行，否則拋出錯誤
- **逾時風險**：單次 user 操作若觸發過多 DB 查詢（~50 次以上），很容易超過 120 秒限制
- **記憶體風險**：大量批量操作時注意 heap 使用量，512MB 看似充裕但大型查詢結果可能快速耗盡
