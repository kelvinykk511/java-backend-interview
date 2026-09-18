# ThreadLocal 與跨執行緒上下文（TTL）

## 面試問題

`ThreadLocal` 解決什麼問題？丟進執行緒池為什麼會出大坑？`TransmittableThreadLocal`（TTL）補哪一塊？

## 答法

- **ThreadLocal**：每個執行緒一份獨立變數副本。典型用途：請求上下文（traceId、userId）、SimpleDateFormat 避共用、事務/安全上下文綁定。
- **生命週期**：用完一定 `remove()`，尤其在 **執行緒池**——執行緒會複用，不清就洩漏或串請求（上一個用戶的 userId 殘留）。
- **子執行緒 / 執行緒池提交**：普通 `ThreadLocal` **不會**自動傳到新執行緒；`InheritableThreadLocal` 只在 **建立執行緒當下** 拷貝一次，對池化複用無效。
- **TTL（阿里 TransmittableThreadLocal）**：在任務提交到執行緒池時捕獲父執行緒上下文，執行前後還原。配合 `TtlExecutors` 包裝池，適合把 traceId 傳到 `@Async` / 業務執行緒池。
- 對標：MDC（Logback）也常靠類似「提交時拷貝」；沒包裝池就斷鏈。

## 常見追問 / 陷阱

「ThreadLocal 會造成記憶體洩漏嗎？」

→ Entry 的 key 是弱引用，但 **value 是強引用**。執行緒池執行緒長存 → value 不 `remove` 就一直佔著。面試標準答案：**池化場景必須 finally remove**。
