# ForkJoinPool 工作竊取 + parallelStream 陷阱

## 面試題
`parallelStream()` 底層係乜？點解生產環境好多人禁用佢？

## 答案要點
- **底層**：`ForkJoinPool.commonPool()`，全 JVM 共用；預設並行度 = `CPU 核數 - 1`（調用線程自己都會幫手做）。
- **工作竊取（work stealing）**：每個 worker 有自己嘅**雙端隊列**；自己由頭（LIFO）攞任務，做完就去**偷**其他 worker 隊尾（FIFO）嘅任務 → 減少競爭、大任務拆細後負載自動平衡。
- **適合**：CPU 密集、可以遞歸拆分（`RecursiveTask` / `RecursiveAction`，fork 一半、自己 compute 另一半、再 join）、數據量大、數據源易拆（`ArrayList`、數組；`LinkedList`、`Stream.iterate` 拆得差）。
- **禁用原因**：
  1. **共用池**：一個請求入面 `parallelStream` 做 I/O（查 DB / 調 HTTP），阻塞咗 commonPool 幾條線程 → 全 JVM 所有 `parallelStream`、預設 `CompletableFuture.supplyAsync` 一齊慢。
  2. 線程數細（核數 - 1），I/O 任務根本唔夠用。
  3. **線程安全**：喺 `forEach` 入面 `list.add` 到 `ArrayList` → 丟數據 / 越界；要用 `collect`。
  4. **ThreadLocal / MDC / 事務 / 安全上下文**唔會傳到 worker 線程（traceId 斷、`@Transactional` 唔生效）。
  5. 數據少時拆分 + 合併開銷大過收益，反而慢。
- **做法**：I/O 用自己嘅業務線程池 + `CompletableFuture`（或虛擬線程）；真係要 CPU 並行就自建 `new ForkJoinPool(n)` 隔離，唔好搶 commonPool。

## 常見追問 / 陷阱
- **「`pool.submit(() -> list.parallelStream()...)` 就會用自定義池？」**：實際上會（任務喺該 FJP 入面 fork），但呢個係實現細節唔係規範保證，好多人當 trick 用；更穩陣係唔用 parallelStream 做 I/O。
- **`join()` 喺 FJP 入面阻塞**：FJP 會用 compensation 補線程或者幫手執行其他任務；但如果你喺 worker 入面阻塞喺**外部** I/O，佢唔知 → 冇補償，池就餓死。需要可以用 `ManagedBlocker`。
- `-Djava.util.concurrent.ForkJoinPool.common.parallelism=N` 可以改 commonPool 大小，但係全局影響，治標唔治本。
- 虛擬線程嘅預設調度器都係一個（獨立嘅）ForkJoinPool，FIFO 模式。
