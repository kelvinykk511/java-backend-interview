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

---

## 深入（2026-11-17）：超時預算、空閒 vs 總時長、重試放大

### 面試可以點追

調風控／簽名機時，connect、read、response／call、池 acquire 分別擋什麼？read 超時設得比接口 SLA 長、超時後再自動重試，會發生咩事？

### 答法

**四層仍然要分清，但語意唔好混**

- **Connect**：TCP + 多數情況包含 TLS 握手。對端掛掉、SYN 黑洞、證書協商慢，靠呢層切斷。
- **Read／Socket timeout**：好多客戶端（Apache `SO_TIMEOUT`、OkHttp `readTimeout`）係「兩個 byte 之間最長空閒」，**唔係**整段 response 嘅總時長。對端慢慢滴數據，可以拖過 SLA 都唔觸發。
- **Response／call／request timeout**（OkHttp `callTimeout`、JDK `HttpRequest.timeout`、WebClient `responseTimeout`）：先當「呢一次呼叫嘅總預算」。具體起算點因客戶端而異，所以要對住你用嗰款 SDK 文件核對，唔好假設名相同就語意相同。
- **Pool acquire／connection request**：池入面借連線嘅等待上限。池耗盡時線程卡喺呢度，監控會以為「下游慢」，其實下游可能根本未接到新請求。

**對齊方法**

- 口訣不變：業務 SLA > 讀／response 預算 > 取連線 ≥ 連線超時。外層（網關／Controller）已經 1s 要回，內層 read 2s 等於用戶已 504，你嘅 Tomcat 線程仲佔住等。
- 調下游要預留自己嘅計算同其他依賴時間，唔好每個依賴都用滿成個 SLA。
- DNS 解析喺部分客戶端**唔計入** connect timeout；解析慢或 `InetAddress` 快取咗壞 IP，會表現成偶發超時。連線複用（keep-alive）係另一題，但「每次 new 客戶端」會令 connect／TLS 次次都付，P99 極差。

**超時之後點做**

- 冪等讀（查行情、查單）先可有限重試，而且要有總預算（3 次 × 2s ≠ 還係 2s）。
- 下單／出金唔好盲目重試：對端可能已成功、只係 response 丟失。靠冪等鍵／狀態機，重試先安全。
- 超時 ≠ 熔斷：超時切單次等待；熔斷用失敗率／慢呼叫比例，避免繼續打已病嘅下游。兩者要一齊設，光有超時仍然會用線程去「等死」。

### 追問／陷阱

- 「P99 差就調大超時」：通常唔好。更多線程堵更久，雪崩放大。先查池是否排隊、acquire 有冇超時、對端 P99、依賴鏈有冇層層疊加。
- **Retry storm**：超時 2s 再自動重試 3 次，一次用戶請求最多打下游 4 次、佔線程最多約 8s。下游越慢，重試越幫佢加流量。重試次數、總預算、指數退避同熔斷要一齊設計。
- JDK `HttpClient` 常用就係 connect + 一個 request timeout，**唔等於**你已經分層設好 read／acquire；WebClient 唔設 responseTimeout，底層可能一直等。
