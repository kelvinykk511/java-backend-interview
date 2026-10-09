# HTTP 客戶端超時設置

## 面試問題

調下游 HTTP（HttpClient / OkHttp / WebClient）時，常見有哪幾種超時？分別保護什麼？設錯會怎樣？

## 答法

至少分清這四層（名稱因客戶端而異，語意共通）：

1. **Connect timeout**：建 TCP（+ 常含 TLS 握手）的上限。保護對端掛掉、網路不通、SYN 黑洞。
2. **Socket / Read / Response timeout**：連上之後，等資料的上限（讀下一個 byte 或整段 response，看實作）。保護對端處理極慢、半開連線。
3. **Write / Request timeout**：寫 request body 的上限。大上傳、對端收得很慢時用得上。
4. **Connection request / Pool acquire timeout**：從連線池「借」一條連線的等待上限。保護池耗盡時執行緒無限卡死。

實務口訣（CEX 調風控／清算常見）：

- **業務超時 > 讀超時 > 取連線超時 ≥ 連線超時** 一條鏈對齊；取連線超時若長過業務超時，業務先爆、線程還在池前傻等。
- 只設 connect、不設 read → 慢對端拖死 Tomcat／業務線程池（看起來像本服務卡死）。
- 超時後要有明確策略：幂等 GET 可重試；非幂等寫要靠冪等鍵，不能盲目重試。
- 超時 ≠ 熔斷：超時是單次請求天花板；熔斷看失敗率／慢呼叫比例，避免繼續打已病的下游。

## 常見追問 / 陷阱

「P99 變差，是不是把超時調大就好？」

→ 多數情況**不要**。調大只是讓更多線程堵更久，放大雪崩。先查：池是否排隊、對端 P99、GC、依賴鏈超時是否層層疊加（A 等 B 等 C）。寧可短超時 + 降級／熔斷，也不要無限等。

關聯：[HTTP Keep-Alive 與連線池](http-keepalive-connection-pool.md)、[熔斷／降級](../distributed/circuit-breaker-degradation.md)
