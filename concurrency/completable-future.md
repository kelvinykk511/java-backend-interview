# CompletableFuture：異步編排

## 面試問題

`CompletableFuture` 比裸 `Future` 強在哪？`thenApply`／`thenCompose`／`thenCombine` 怎麼選？執行緒池用錯會有什麼坑？

## 模範答案（面試可講）

- `Future` 基本只能 `get` 阻塞；`CompletableFuture` 支援**回調鏈**、組合多個異步、顯式完成／異常，適合後端並行查下游、聚合結果。
- `thenApply`：同步轉換結果 `T → U`（還在同一階段的「映射」）。
- `thenCompose`：扁平化「回傳另一個 `CompletionStage`」——避免 `CompletableFuture<CompletableFuture<U>>`（類似 `flatMap`）。
- `thenCombine`／`allOf`／`anyOf`：組合**兩個或多個**獨立 Future；匯總時注意例外與取消。
- 預設很多回調跑在 **`ForkJoinPool.commonPool()`**（或完成該階段的執行緒）；I/O／呼叫下游應自備 **業務執行緒池**，用 `*Async(..., executor)`，避免佔滿 commonPool、也避免在 Tomcat 請求執行緒上阻塞 `join`／`get`。
- 異常：`handle`／`whenComplete`／`exceptionally`；別只 `get()` 不處理 `CompletionException` 包裹。

## 常見追問／陷阱

- 在並行流／CF 裡再提交到同一個會阻塞的池 → **死鎖或執行緒耗盡**。
- `allOf(...).join()` 後還要對每個 CF `join` 才能拿到結果並正確浮出例外。
- 「開了虛擬執行緒就能亂 `get`」仍要談超時、取消與下游限流——面試要講清**背壓與隔離池**。

## 關聯

- [ThreadPoolExecutor](thread-pool-executor.md)
