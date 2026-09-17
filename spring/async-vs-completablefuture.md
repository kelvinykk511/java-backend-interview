# @Async 與 CompletableFuture

## 面試問題

Spring `@Async` 和自己用 `CompletableFuture` / 線程池，分別什麼時候用？有什麼坑？

## 答法

- **`@Async`**：方法級「丟到別的線程跑」，適合簡單異步（發通知、寫審計）。要開 `@EnableAsync`，並建議自訂 `Executor`（別用默認簡單池）。
- **`CompletableFuture`**：要編排（thenCompose / thenCombine / allOf）、超時、異常鏈時更合適；和業務代碼綁在一起，可控性強。
- **相同點**：都要明確線程池、拒絕策略、MDC/TraceId 傳遞；都別在請求線程裡盲目 `join()` 堵死。
- **面試一句話**：簡單「火後不管」用 `@Async`；要組合多步異步、對齊超時與錯誤處理，用 `CompletableFuture` + 自訂池。

## 常見追問 / 陷阱

「同類別裡 `this.asyncMethod()` 會異步嗎？」

→ 通常**不會**。`@Async` 靠代理，自調用繞過代理（跟 `@Transactional` 同類坑）。要抽到別的 Bean，或顯式用注入的代理 / `ApplicationContext`。

## 小練習

下單成功後要：寫庫 → 發 Kafka → 調風控 HTTP。哪幾步適合 `@Async`，哪幾步該用 `CompletableFuture` 並設超時？為什麼？
