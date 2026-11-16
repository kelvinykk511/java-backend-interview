# Spring MVC vs WebFlux 執行緒模型

## 面試問題

Spring MVC（Servlet）和 WebFlux 的執行緒模型差在哪？什麼場景該用 WebFlux，什麼場景用 MVC 就夠？

## 答法

**MVC（Servlet / Tomcat 典型）**

- 一請求一（或少數）**工作執行緒**跑完整條調用鏈。
- 調下游 HTTP/DB 若是**同步阻塞**，執行緒就佔着等——併發上限≈執行緒池大小。
- 模型簡單，和現有 JDBC、事務、ThreadLocal 生態契合；大多數業務後端首選。

**WebFlux（Reactive / Netty）**

- 少量 **Event Loop** 執行緒處理大量連接；強調**非阻塞** I/O。
- 在 loop 裡做阻塞調用（同步 JDBC、`Thread.sleep`、重 CPU）會**拖死**整條 loop——要 `publishOn`/`subscribeOn` 把阻塞丟到專用池，或根本別混用。
- 適合：高併發、大量 I/O 等待、閘道/代理、流式（SSE）且整條鏈可 reactive。

**怎麼選（面試口徑）**

| | MVC | WebFlux |
|--|-----|---------|
| 生態 | JDBC、`@Transactional`、同步 SDK 成熟 | 要 reactive 驅動 / 非阻塞客戶端 |
| 心智負擔 | 低 | 高（背壓、調度、調試難） |
| 典型場景 | CRUD、交易核心、複雜事務 | API Gateway、長連接推送、純 I/O 密集 |

口訣：**不是「WebFlux 一定更快」**；阻塞比例高時 MVC + 夠用的池往往更穩。Netty 那套「別堵 EventLoop」和 WebFlux 是同一思維。

## 常見追問 / 陷阱

「上了 WebFlux 再用阻塞 Feign/JDBC，是不是還是響應式？」

→ 名義上是 WebFlux，實際上把 Event Loop 堵死，**比 MVC 更糟**（loop 執行緒更少）。要嘛全鏈非阻塞，要嘛老老實實 MVC。

## 小練習

CEX 下單接口要走本地事務 + 同步寫庫 + 調風控 HTTP：選 MVC 還是 WebFlux？若要做行情 WebSocket 廣播呢？

---

## 深入（2026-11-16）：線程佔用、虛擬線程、同服務拆分

### 面試可以點追

CEX 同時有「下單」（本地事務 + JDBC + 同步風控 HTTP）同「行情推送」（大量連線、I/O 等待）。Spring MVC 同 WebFlux 線程模型差在邊？成個 Boot 應用統一改 WebFlux 可唔可以？JDK 21 虛擬線程係咪可以代替 WebFlux？

### 答法

**線程係點被佔用**

- **MVC**：一條請求通常佔住一條 Tomcat worker 由進到出。下游同步 HTTP／JDBC 一慢，線程就佔着等，吞吐 ≈ 池大小 / 平均耗時。Servlet 3 異步（`DeferredResult`／`Callable`）可以提早還容器線程，但業務工作仍要自己嘅池；ThreadLocal、`@Transactional` 仍然係「一條線程綁一段工作」嘅模型。
- **WebFlux**：少量 Event Loop（常見約 CPU 核數）同時掛住大量 in-flight 請求，請求唔綁死一條線程。狀態用 **Reactor Context**，唔好靠 ThreadLocal。數據喺 loop 上到，阻塞或重 CPU 必須 `subscribeOn`／`publishOn` 去專用池，否則拖死呢條 loop 上嘅所有連線。
- MVC 用 WebClient 再 `.block()`，只係換咗 HTTP 客戶端，**線程模型冇變**，Tomcat 線程一樣被佔住。

**點選、點拆**

- 下單／出金／對帳：要本地事務、同步 JDBC、現成同步 SDK → **MVC**。阻塞比例高時，WebFlux 唔會更快。
- 行情 WS、API Gateway、純 I/O 密集而且整條鏈可以非阻塞 → **WebFlux** 或獨立 Netty 進程。兩個模型可以分服務，唔好為「統一技術棧」把事務核心硬改響應式。
- **JDK 21 虛擬線程**令阻塞式 MVC 可以捱更高併發（`spring.threads.virtual.enabled`），好多後端唔使改寫 WebFlux。但 WebFlux 嘅 Event Loop 仍然係平台線程，喺 loop 上 `block()`／同步 JDBC，虛擬線程救唔到；`synchronized` pinning、下游配額仍然要管。虛擬線程係「令阻塞更平」，唔係「把 Event Loop 變成唔會被堵」。

### 追問／陷阱

- 喺 WebFlux handler 入面 `webClient.get()...block()`、同步 Feign／JDBC：名義響應式，實際把**更少**嘅 loop 線程堵死，比 MVC 更糟。阻塞要丟去 `Schedulers.boundedElastic()`（或專用池），而且**唔好喺 Event Loop 上 `.block()`**。
- 「用咗 WebClient 就係 WebFlux」係錯嘅。相反，MVC + 虛擬線程亦唔等於可以喺 WebFlux Event Loop 上隨便阻塞。
- 事務、SecurityContext、MDC 喺 WebFlux 要跟 Reactor Context 傳播；照搬 ThreadLocal 會空、會串。
