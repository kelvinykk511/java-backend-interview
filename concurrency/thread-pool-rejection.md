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

---

## 深入（2026-11-12）：CallerRuns 打中 Tomcat／EventLoop，同「拒絕永遠唔觸發」

### 面試可以點追

「拒絕策略用 `CallerRunsPolicy` 做背壓，係咪一定穩陣？點解有時明明配咗 max 同拒絕策略，線上從來冇 rejected 指標，堆卻先爆？」

### 答法

**CallerRuns 背壓嘅代價係「誰提交，誰跑」**

- 策略本身冇錯：隊列滿就讓**呼叫線程**自己跑，提交自然變慢。
- 但呼叫線程若係 **Tomcat／Netty EventLoop／閘道 I/O 線程**，等於用接收連線嘅線程跑業務。高峰會：請求線程耗盡、EventLoop 心跳停、健康檢查都排唔上。
- 池內執行緒再 `submit().get()`，加上 CallerRuns，有機會死鎖或自我阻塞（任務要等池，池又叫呢條線程跑）。
- CEX 下單／出金：多數選 **AbortPolicy**（或自定義 handler）→ 快速失敗、限流、回 503／可重試錯誤；**唔好**用 Discard／DiscardOldest 默默丟撮合、入帳、出金任務。日誌、可丟嘅 metrics 先考慮 Discard。

**拒絕策略永遠唔觸發 = 無界隊列**

- `Executors.newFixedThreadPool` / `newSingleThreadExecutor`：`LinkedBlockingQueue` 無界 → 任務永遠喺「進隊列」呢一步，**開唔到 max，亦走唔進拒絕**。堆上堆積任務（每個 Future／閉包咬住上下文）→ 延遲爆、最後 OOM。
- 要拒絕先要 **有界隊列**（例如 `ArrayBlockingQueue` 定容量）。`newCachedThreadPool` 則係同步交接 + max 近乎無限，拒絕唔到，改為狂開線程。
- 實務：自己 `new ThreadPoolExecutor`，有界隊列 + 具名 threadFactory + 明確 handler；監控 **隊列深度、活躍線程、rejected 次數**。按工作負載拆池，唔好同 EventLoop／Tomcat 混用一條「萬能池」。

### 追問／陷阱

「CallerRuns 會唔會死鎖？」→ 池內互相 `get()`、或者 EventLoop 既 `get()` 又 CallerRuns，會。業務由 I/O 線程提交時，CallerRuns 先係**把阻塞轉移到入口**，唔係免費容量。另一陷阱：以為配咗 Abort 就安全，但隊列用咗無界 `LinkedBlockingQueue`，Abort 永遠唔會行。
