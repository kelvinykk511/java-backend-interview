# 熔斷與降級（Circuit Breaker / Fallback）

> 姊妹篇：[限流算法](rate-limiting.md)

## 面試問題

下游（例如風控或報價服務）變慢，你的服務線程池被拖滿、全面超時。熔斷怎麼救？熔斷、限流、降級三者有什麼分別？

## 答法

- **三態**：
  - `CLOSED`：正常放行，用滑動窗口（按次數或按時間）統計失敗率、慢調用率。
  - 超過閾值（且達到最小調用數）→ `OPEN`：直接快速失敗、走 fallback，不再打下游。
  - 等一段時間 → `HALF_OPEN`：只放少量探測請求；成功率夠就回 `CLOSED`，不夠再 `OPEN`。
- **三者分工**：
  - **限流**：守入口，保護自己不被上游打爆。
  - **熔斷**：守出口，保護自己不被慢下游拖死，順便讓下游喘口氣。
  - **降級**：觸發後的替代方案，例如返回緩存、默認值、關掉非核心功能。
- **前提**：一定要有**超時**（沒超時，熔斷器根本不知道「慢」）；最好配**隔離（bulkhead）**，每個下游用獨立線程池或信號量，一個下游壞不拖垮全部。
- **Resilience4j** 常用參數：`failureRateThreshold`、`slowCallRateThreshold` + `slowCallDurationThreshold`、`minimumNumberOfCalls`、`slidingWindowSize`、`waitDurationInOpenState`、`permittedNumberOfCallsInHalfOpenState`。（Hystrix 已停止維護，新項目講 Resilience4j / Sentinel。）

## 常見追問／陷阱

- **哪些錯誤算失敗？** 業務錯誤（餘額不足、參數錯、4xx）不該計入，否則用戶亂輸入也能把熔斷打開。用 `recordExceptions` / `ignoreExceptions` 區分。
- **寫操作別亂降級**：查行情可以返回緩存；但下單、出金不能 fallback 成「假成功」。寫操作降級 = 明確失敗（讓用戶重試）或入隊異步處理，並保證冪等。
- **粒度**：按下游、甚至按接口分開熔斷；單個實例壞了應該靠負載均衡摘除，而不是把整個服務熔斷。
- **minimumNumberOfCalls 太小**：低流量時兩三次失敗就熔斷，抖動很大。
