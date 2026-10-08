# CountDownLatch vs CyclicBarrier vs Semaphore

> 面試題：要並行調三個下游服務，全部返回後再組裝結果，你用哪個？CyclicBarrier 可以嗎？

## 一句話區分

| | 誰等誰 | 能否重用 | 底層 |
|---|---|---|---|
| `CountDownLatch` | 一個（或多個）線程等 N 件事做完 | 不能，計數到 0 就結束 | AQS 共享模式 |
| `CyclicBarrier` | N 個線程**互相等**，到齊一起往下走 | 能，到齊後自動重置 | `ReentrantLock` + `Condition` |
| `Semaphore` | 限制同時最多 N 個線程進入 | 能，借還許可 | AQS 共享模式 |

## CountDownLatch

- `countDown()` 減一（不阻塞），`await()` 等到 0。
- 典型：主線程等 N 個子任務完成；服務啟動等依賴就緒。
- 坑：子任務拋異常沒 `countDown` → 主線程永遠等。**`countDown()` 放 finally**，`await(timeout, unit)` 帶超時。
- 現代寫法：並行調下游更推薦 `CompletableFuture.allOf(...)`，能拿結果、處理異常、帶超時。

## CyclicBarrier

- N 個參與者都調 `await()` 才一起放行，可帶 `barrierAction`（最後到達的線程執行）。
- 典型：多線程分段計算，每輪全部算完再進下一輪（多輪＝cyclic）。
- 坑：一個線程中斷或超時 → barrier 變 **broken**，其他等待者拋 `BrokenBarrierException`；要 `reset()`。
- 坑：參與者數 > 線程池線程數 → 部分任務排隊進不來，已進的永遠等不到人 → 死等。

## Semaphore

- `acquire()` 拿許可、`release()` 還；`tryAcquire(timeout)` 拿不到可快速失敗。
- 典型：限制調某下游的並發數（艙壁/bulkhead）、資源池。
- 坑：`release()` 不在 finally → 許可洩漏，慢慢變成全部阻塞。
- 坑：`release()` 不檢查是否拿過，多 release 會讓許可**變多**。
- 這是**單機**並發限制；集群限流要用 Redis/網關。

## 回答開頭的題

- 用 `CountDownLatch`（或 `CompletableFuture.allOf`）：主線程等三件事做完。
- `CyclicBarrier` 不合適：它是讓「工作線程彼此等」，主線程不是參與者；而且調下游的線程到齊後也沒有「一起往下做」的需要。
