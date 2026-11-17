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

---

## 深入（2026-11-17）：重試同熔斷嘅疊加、共用 name、HALF_OPEN

### 面試可以點追

已經設咗 HTTP 超時，點解仲要熔斷？Retry、CircuitBreaker 包嘅次序會點影響開閘速度同線程佔用？幾個接口共用一個 breaker name 有咩問題？

### 答法

**超時同熔斷分工**

- 超時：單次呼叫嘅天花板，避免一條請求無限等。
- 熔斷：看一窗口內嘅失敗率／慢呼叫比例（要有 `minimumNumberOfCalls`），超過就 **OPEN**，後續請求 fail-fast 走 fallback，唔再佔線程打已病下游，亦畀下游喘息。
- 冇超時，慢呼叫統計幾乎唔會準，線程照樣被堵死，熔斷救唔到。最好再加 **bulkhead**（每下游獨立線程池或信號量），一個風控慢唔好拖垮出金。

**同 Retry 一齊用，次序好關鍵**

- `CircuitBreaker(Retry(call))`：要等內部重試全部走完，先當一次成功／失敗。熔斷開得慢，線程被重試佔更久，下游喺呢段時間被打得更狠。
- `Retry(CircuitBreaker(call))`：每次嘗試都計入熔斷；一旦 OPEN，重試都會 fail-fast，唔再打下游。通常更合理。
- 無論邊種，業務 4xx／「餘額不足」都應該排除出失敗統計同重試條件，否則用戶輸入就可以把閘打開。

**降級仍然要分讀寫**

- 查行情／查餘額：可以返回短 TTL 緩存或明確「暫時不可用」。
- 下單／出金：只能明確失敗或入隊異步（狀態機 + 冪等），**禁止** fallback 成假成功。
- Fallback 若再打「備援 URL」但其實係同一個集群 VIP，等於冇斷。

### 追問／陷阱

- **共用 CircuitBreaker name**：行情查詢同下單共用一個實例，行情一抖，出金亦被 fail-fast。按下游、按接口分開；單個實例壞應靠 LB 摘除，唔好把成個服務熔斷。
- **HALF_OPEN 放量過大**：`permittedNumberOfCallsInHalfOpenState` 設 50，等於一開閘就用探針把下游再打一輪。探針要少，成功先逐步放行。
- **時間窗口 vs 計數窗口**：低流量時 `minimumNumberOfCalls` 太細，兩三次失敗就 OPEN，抖動好大；發佈後 JIT／連接池預熱令 P99 短暫變差，慢呼叫閾值太緊會誤熔斷（放量、排除發佈窗口、或預熱）。
- 熔斷 OPEN 唔等於本服務健康：入口限流仲要守住；對端恢復後要靠 HALF_OPEN 探測，唔好人工永久關閘就當修咗。
