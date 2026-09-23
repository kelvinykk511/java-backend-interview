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
