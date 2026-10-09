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
