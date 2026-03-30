---
name: javacat-todojob-mechanism
description: >
  JavaCat 待辦工作（TodoJob）執行機制完整解析。涵蓋 Worker 排程架構、佇列優先權調度、
  狀態機流轉（IDLE/WORKING/PENDING/ERROR/DONE/CANCELLED）、WORKING 遞迴執行機制、
  keepQueue 佇列控制、buffer 跨階段傳遞、錯誤重試與取消策略、以及交易 commit/rollback 行為。
  當需要撰寫、除錯、或理解任何 TodoJob 函數腳本時，務必先讀此 skill。
  適用場景包括：撰寫新的待辦工作函數、理解現有待辦工作的執行流程、排查待辦工作卡住或重複執行問題、
  設計多階段批次處理流程。
---

# JavaCat 待辦工作（TodoJob）執行機制

TodoJob 是 JavaCat 中的異步任務處理系統，透過 AWS Lambda 定期觸發 Worker，從佇列中取出待辦工作並執行對應的表格函數腳本。

## 參考文件目錄

| 章節 | 內容摘要 | 何時閱讀 |
|------|---------|---------|
| [architecture-overview.md](references/architecture-overview.md) | Worker 排程架構、佇列優先權調度、Lambda 觸發流程 | 第一次了解 TodoJob 系統 |
| [status-lifecycle.md](references/status-lifecycle.md) | 六種狀態定義、狀態流轉規則、各狀態對移除/保留的影響 | 理解待辦工作的生命週期 |
| [working-recursion.md](references/working-recursion.md) | WORKING 遞迴執行機制、saftyKeepTodoJobWorking 安全工具、多階段處理模式 | 撰寫多階段待辦工作函數 |
| [keepqueue-mechanism.md](references/keepqueue-mechanism.md) | keepQueue 預設值邏輯、與狀態的交互、佇列流控策略 | 控制佇列是否繼續執行 |
| [buffer-mechanism.md](references/buffer-mechanism.md) | buffer 傳遞流程、序列化限制、多階段 stage 設計模式 | 設計跨執行階段的資料傳遞 |
| [error-retry-strategy.md](references/error-retry-strategy.md) | 錯誤重試、onJobRetry 設定、execCount 上限、commit/rollback 行為 | 理解錯誤處理與重試策略 |
| [writing-todojob-functions.md](references/writing-todojob-functions.md) | targetName → record 對應關係、Upsert 模式、ctx.todoJob.body.param 傳參、session:true 冪等、queueNo 分配防 race condition | **撰寫任何新的 TodoJob 函數腳本前必讀** |

## 快速定位

- **撰寫新的待辦工作函數** → **先讀 [writing-todojob-functions.md](references/writing-todojob-functions.md)**，再讀 [status-lifecycle.md](references/status-lifecycle.md)
- **目標記錄可能不存在（upsert）** → [writing-todojob-functions.md](references/writing-todojob-functions.md)
- **設計多階段批次處理** → [working-recursion.md](references/working-recursion.md) + [buffer-mechanism.md](references/buffer-mechanism.md)
- **控制佇列執行行為** → [keepqueue-mechanism.md](references/keepqueue-mechanism.md)
- **排查錯誤或卡住問題** → [status-lifecycle.md](references/status-lifecycle.md) + [error-retry-strategy.md](references/error-retry-strategy.md)

## 核心概念速覽

### 系統架構

```
AWS EventBridge (定時觸發)
  → Lambda (todoJobWorker)
    → MyTodoJobScheduleService
      → getJobQueueList()        取得所有工作佇列
      → genQueueSchedule()       依優先權排序（加權隨機）
      → for each queue:
          → tryMarkJobQueueAsRunning()  佔用佇列（樂觀鎖）
          → while (未超時):
              → fetchOneTodoJob()       取出一筆工作
              → executeOneTodoJob()     在 Sandbox 中執行函數
              → 根據回傳狀態決定移除/保留/繼續
```

### 待辦工作資料結構

```typescript
TodoJobRecord {
  body: {
    name: string;           // 工作編號
    tableName: string;      // 所屬表格
    functionName: string;   // 對應函數名稱
    status: TodoJobStatus;  // 執行狀態
    queueNo: number;        // 佇列編號
    execCount: number;      // 累計執行次數
    errorCount: number;     // 連續錯誤次數
    buffer?: string;        // 跨階段暫存區
    param?: string;         // 自訂參數
    nextPickAt?: Date;      // 下次取用時間（PENDING 用）
  }
  lines: {
    items: []                          // 模式一：空陣列，無目標記錄（批次處理複數記錄）
    // 或
    items: [{ targetName: string }]    // 模式二：指定單一目標（僅限一個，不可兩個以上）
  }
}
```

### 函數回傳型別

```typescript
TodoJobFunctionReturn {
  status: TodoJobStatus;      // 必填：決定工作的下一步
  result?: string;            // 執行結果（記錄在工作歷程）
  buffer?: string;            // 傳遞給下次執行的暫存資料
  keepQueue?: boolean;        // 是否繼續執行佇列中的下一個工作
  commit?: boolean;           // 控制資料庫 commit/rollback
  delaySeconds?: number;      // PENDING 狀態下的延遲秒數
}
```
