# 線程池拒絕策略與 Executors 陷阱

## 面試問題

`ThreadPoolExecutor` 佇列滿且執行緒到頂時會怎樣？四種拒絕策略差在哪？為什麼生產少用 `Executors.newFixedThreadPool`？

## 答法

任務進池順序先複習一句：未滿 core → 開執行緒；滿 core → 進佇列；佇列滿 → 開到 max；還進不來 → **RejectedExecutionHandler**。

| 策略 | 行為 | 何時用 |
|------|------|--------|
| **AbortPolicy**（預設） | 丟 `RejectedExecutionException` | 要快速失敗、由上層重試／降級 |
| **CallerRunsPolicy** | 呼叫執行緒自己跑這任務 | 自然背壓，減慢提交速度 |
| **DiscardPolicy** | 默默丟棄新任務 | 可丟的日誌／指標類（慎用） |
| **DiscardOldestPolicy** | 丟佇列頭再試 | 只要最新結果的場景（也要慎） |

**Executors 工廠陷阱（常考）**

- `newFixedThreadPool` / `newSingleThreadExecutor`：佇列是 **無界** `LinkedBlockingQueue` → 永遠開不到 max，任務堆在堆上，易 **OOM**。
- `newCachedThreadPool`：同步佇列 + max=Integer.MAX_VALUE → 突刺時狂開執行緒。

實務：自己 `new ThreadPoolExecutor`，**有界佇列** + 明確拒絕／降級 + 執行緒命名；監控佇列深度與拒絕次數。

## 常見追問／陷阱

「CallerRuns 會不會死鎖？」→ 若呼叫執行緒是池內執行緒且任務又互相 `submit().get()`，有可能；業務提交執行緒一般是 Tomcat／網關執行緒，多半是背壓而非死鎖，但仍要避免池內同步互等。
