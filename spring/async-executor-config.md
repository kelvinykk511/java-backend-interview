# Spring @Async 線程池配置

## 面試問題

開了 `@EnableAsync` 之後，預設用哪個執行器？線上該怎麼配 `@Async` 的線程池？有哪些必踩坑？

## 答法

- 未自訂時，Spring 常用 **SimpleAsyncTaskExecutor**：幾乎**每個任務新建執行緒**，無上限、無複用 → 高併發可把機器打爆。面試必講：生產一定要自訂 `TaskExecutor` / `AsyncConfigurer`。
- 自訂要點（對齊 `ThreadPoolExecutor`）：
  - **core / max / queue**：I/O 密集可多一點線程；CPU 密集 ≈ CPU 核數；隊列有界，避免 OOM。
  - **拒絕策略**：業務異步常用 `CallerRunsPolicy`（回壓到呼叫端）或丟棄+打點告警；別默默 `AbortPolicy` 然後沒人接異常。
  - **執行緒名前綴**：方便 jstack／日誌。
  - **裝飾器**：傳遞 MDC / TraceId（否則異步日誌斷鏈）。
  - **異常處理**：`AsyncUncaughtExceptionHandler` 接 void `@Async` 的未捕獲異常；有回傳值用 `Future` / `CompletableFuture` 自己處理。
- 與請求執行緒關係：Controller 裡對 `@Async` 的 `Future.get()` 若無超時，等於把 Tomcat 線程堵回去，失去異步意義。

面試一句話：`@Async` 只是把方法丟進你配好的池；池的大小、隊列、拒絕策略、上下文傳遞，才是生產問題的答案。

## 常見追問 / 陷阱

1. **同類自調用不生效**（跟 `@Transactional` 一樣靠代理）→ 抽到別的 Bean。
2. **多個 Executor**：用 `@Async("auditExecutor")` 指名；別全部擠一個池，避免審計任務拖垮核心異步。
3. **優雅停機**：要等池收尾（`spring.task.execution.shutdown.await-termination` 或自行 `DisposableBean`），否則進程殺了半截任務。

關聯：[@Async 與 CompletableFuture](async-vs-completablefuture.md)、[線程池拒絕策略](../concurrency/thread-pool-rejection.md)、[優雅停機](graceful-shutdown.md)

---

## 深入（2026-11-10）：多池拆分、事務邊界、MDC、同優雅停機

### 面試問題

生產開咗 `@EnableAsync`，有人用默認執行器、有人喺 `@Transactional` 方法裡直接 `@Async` 發消息、高峰時線程暴漲仲有 TraceId 斷鏈。你會點設計線程池同調用方式？

### 答法

**1. 默認執行器係坑**

- 未自訂時常見 **SimpleAsyncTaskExecutor**：幾乎**一任務一新線程**，無界 → 流量一湧就打爆記憶體／FD／CPU。
- 生產必備：實現 `AsyncConfigurer`（或定義名為 `taskExecutor`／`applicationTaskExecutor` 嘅 `ThreadPoolTaskExecutor`），**有界隊列 + 合理 max + 拒絕策略 + 線程名前綴**。

**2. 多池拆分（CEX 後端常問）**

| 池 | 用途 | 點解分開 |
|----|------|----------|
| `bizExecutor` | 成交回報、賬務異步 | 核心路徑，要背壓清晰 |
| `ioExecutor` | 調簽名機、HTTP 風控 | I/O 密集，core／max 可以大啲 |
| `auditExecutor` | 審計日誌、非關鍵通知 | 避免拖垮核心池 |

用 `@Async("ioExecutor")` 指名；拒絕策略：核心業務偏好 `CallerRunsPolicy`（回壓）+ 打點；審計可用丟棄+告警，但要監控。

**3. 事務邊界（高頻陷阱）**

- `@Async` 方法跑喺**別嘅線程**，同呼叫方**唔共享**同一個 Spring 事務上下文。
- 喺 `@Transactional` 方法裡「先改 DB、再 `@Async` 發 MQ／通知」：異步可能喺**事務未 commit** 就跑，讀到舊數據或發咗事後又 rollback → 用 **TransactionSynchronization `afterCommit`** 再提交異步任務，或者發 Outbox，等事務提交後再異步。
- `@Async` + `@Transactional` 標喺同一個方法：事務係喺異步線程開，語義要講清楚，唔好假設同 Controller 一體。

**4. 上下文傳遞**

- 線程池要配 **TaskDecorator**：複製 MDC／TraceId／租戶，否則日誌同鏈路斷。
- `void @Async`：靠 `AsyncUncaughtExceptionHandler`；有返回值用 `CompletableFuture`，呼叫端要處理異常同超時。

**5. 優雅停機**

- 池要 `setWaitForTasksToCompleteOnShutdown(true)` + `setAwaitTerminationSeconds`；Boot 亦可配 `spring.task.execution.shutdown.*`。
- 對齊 [優雅停機](graceful-shutdown.md) 嘅時間預算：唔等池收尾 → SIGTERM 後半截任務被砍，消息／出金場景要靠冪等兜底。

### 追問／陷阱

1. **同類自調用**：`this.asyncMethod()` 唔經代理 → `@Async` 失效（同 `@Transactional`）。
2. **Controller 裡 `future.get()` 無超時**：Tomcat 線程堵死，異步變同步，池再大都救唔到入口。
3. **一個巨型共用池**：審計洪峰拖垮出金回調；要隔離同限流。

### 關聯

- [@Async 與 CompletableFuture](async-vs-completablefuture.md)
- [線程池拒絕策略](../concurrency/thread-pool-rejection.md)
- [優雅停機](graceful-shutdown.md)
- [Transactional Outbox](../distributed/transactional-outbox.md)
