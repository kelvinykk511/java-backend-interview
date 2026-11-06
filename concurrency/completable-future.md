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

---

## 深入（2026-11-06）：組合、異常恢復與線程池陷阱

### 面試問題

並行查三個下游要聚合結果，`thenApply` / `thenCompose` / `thenCombine` / `allOf` 點揀？異常用 `exceptionally` 回 `null` 有咩坑？I/O 任務唔指定 executor 會點？

### 答法

**組合點揀**

- **`thenApply(fn)`**：同步映射 `T → U`，`fn` 唔應該再起異步。
- **`thenCompose(fn)`**：`fn` 回另一個 `CompletionStage<U>`（**flatMap**）——鏈式依賴下游（A 結果決定 B 請求）用呢個，避免 `CF<CF<U>>`。
- **`thenCombine(other, fn)`**：兩個獨立 CF 都完成後合併；只有兩個時比 `allOf` 直接。
- **`allOf(cfs...)`**：等全部完成，回 `CompletableFuture<Void>`——**本身冇結果**，之後要對每個 CF `join()` / `get()` 先攞到值；任一失敗，`allOf` 完成時以異常結束，之後 `join` 會拋 `CompletionException`（cause 係原異常）。
- **`anyOf`**：第一個完成就完（成功或失敗都算）；其餘唔會自動 cancel，要自己管。

**默認跑邊個池**

- `supplyAsync(supplier)` **無 executor** → `ForkJoinPool.commonPool()`。
- 非 `*Async` 嘅 `thenApply` / `thenCompose`：跑喺**完成上一階段嗰條線程**（可能係 commonPool worker，也可能係剛好 `complete` 嗰條）。
- `thenApplyAsync` / `thenComposeAsync` 無 executor → 一樣 commonPool。
- **I/O、RPC、DB 必須 `*Async(..., 業務池)`**，唔好污染 commonPool（parallelStream / 其他 CF 共用）。業務池要有界 + 拒絕策略，同 Tomcat 請求池**隔離**。

**異常三件套**

| API | 行為 |
|---|---|
| `exceptionally(fn)` | 只喺異常時跑；`fn` 回傳值變成**正常結果**（「恢復」）。亂回 `null` → 下游當成功，NPE / 默默錯帳。 |
| `handle((r, ex) -> …)` | 成功／失敗都跑；可轉換結果或再拋。 |
| `whenComplete((r, ex) -> …)` | 副作用（日誌、metrics）；**唔改變**結果／異常。 |

- `join()`：失敗拋 **unchecked** `CompletionException`，唔使 catch checked；`get()` 拋 `ExecutionException` / `InterruptedException`。
- **喺 Tomcat 請求線程 `allOf(...).join()` / `get()`**：請求線程被卡住 → 併發一高就**線程池耗盡**、整站拖死。要嘛純響應式鏈式唔阻塞請求線程，要嘛用獨立業務池做供應、設超時，請求線程只短等或轉 async servlet。

### 追問／陷阱

- **`supplyAsync(a).thenApply(b).thenCompose(c)` 邊段用邊個池？**  
  `a` 喺 commonPool（若無指定）；`b`（非 Async）喺跑完 `a` 嗰條線程同步執行；`c` 若 `thenCompose` 非 Async，`c` 嘅 supplier／內部 CF 取決於你喺 `c` 入面點寫——若 `c` 入面又 `supplyAsync` 無池，又係 commonPool。想全程可控：**每一段 I/O 都 `*Async(..., exec)`**。
- **`orTimeout(n, unit)`**（Java 9+）：超時以 **`TimeoutException`** 完成（異常），鏈會失敗。  
  **`completeOnTimeout(value, n, unit)`**：超時用默認值**正常完成**——同樣有「假成功」風險，要同業務語義對齊。
- `exceptionally` 回 `null` + 後面當成功用 → 聚合接口「HTTP 200 但錢包係 null」；應回 `Optional`、明確錯誤碼，或讓失敗向上拋、由統一異常處理。
- `allOf` 一個失敗唔代表其餘會 cancel；仍可能佔住連接／線程，要配合超時同取消。

## 關聯

- [ThreadPoolExecutor](thread-pool-executor.md)
- [虛擬線程](virtual-threads.md)
