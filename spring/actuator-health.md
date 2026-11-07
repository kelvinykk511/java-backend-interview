# Spring Boot Actuator：Health

## 面試問題

Spring Boot Actuator 的 `/actuator/health` 在做什麼？生產環境怎麼控暴露面？自訂一個依賴 DB／Redis 的健康檢查要注意什麼？

## 模範答案（面試可講）

- Actuator 提供**運維可觀測性**端點：`health`、`info`、`metrics`、`prometheus` 等；`health` 彙總各 `HealthIndicator` 狀態（`UP`／`DOWN`／`OUT_OF_SERVICE`／`UNKNOWN`）。
- 預設常只對外顯示 **整體 status**；細節（DB、disk、redis 元件）需開 `management.endpoint.health.show-details`（`when_authorized` 或 `always`）並配好安全。
- 暴露面：用 `management.endpoints.web.exposure.include` **白名單**；生產勿把 `env`／`heapdump`／`shutdown` 隨便公開，通常掛 **獨立 management port** 或內網／閘道認證。
- 自訂：實作 `HealthIndicator`（或 Reactive 版），探 DB／Redis／下游；失敗回 `DOWN` 並帶簡短細節，**不要**把連線字串、密碼寫進 response。
- K8s：`liveness` 與 `readiness` 可拆（Probe 組態／分組），避免「依賴抖一下就殺 Pod」或「還沒就緒就收流量」。

## 常見追問／陷阱

- Health 變 `DOWN` 是否等於要重啟？不一定——可能是依賴短暫失敗；liveness 亂綁依賴會造成**重啟風暴**。
- 「開了 Actuator 就等於監控齊了」不夠：還要 metrics、日誌、追蹤；health 只是存活／就緒訊號。

---

## 深入（2026-11-07）：Liveness vs Readiness、依賴降級、超時與暴露面

### 面試問題

K8s 探活用 Actuator 時，liveness 同 readiness 點分？下游 Redis／DB 短暫 DOWN，點解唔好令 Pod 被殺？健康檢查本身超時或拖慢請求線程點處理？management 端點生產點收暴露面？

### 答法

**Liveness vs Readiness（面試主線）**

| 探針 | 問嘅係咩 | 失敗時 K8s 做咩 | 應不應該綁外部依賴 |
|------|----------|-----------------|-------------------|
| **Liveness** | 進程本身仲活唔活（死鎖、卡住） | 殺 Pod、重啟 | **唔好**綁 Redis／DB／下游；只反映「應用自己仲正常」 |
| **Readiness** | 可唔可以收流量 | 從 Service Endpoints **摘流量**，唔殺 | 可以反映關鍵依賴；依賴抖 → 暫時唔收新流量 |

- Spring Boot 2.3+：`/actuator/health/liveness`、`/actuator/health/readiness`（AvailabilityState）；或用 `management.endpoint.health.group.liveness/readiness` 自訂 include 邊啲 indicator。
- 口訣：**Liveness = 殺重啟；Readiness = 摘流量。依賴抖用 readiness 摘流量，唔好用 liveness 殺 Pod。**

**依賴 DOWN 時點降級（唔好殺 Pod）**

- Redis 抖 2 秒：若 liveness 綁咗 Redis → Pod 被殺 → 滾動重啟 → 其他 Pod 壓力更大 → **重啟風暴**。
- 正確：liveness 只睇進程／本地狀態；readiness 可以因為 Redis DOWN 變 `OUT_OF_SERVICE`／`DOWN`，流量摘走；業務側用**熔斷／降級／快取兜底**（見 [熔斷降級](../distributed/circuit-breaker-degradation.md)），等依賴恢復再重新就緒。
- 非關鍵依賴（例如非必須嘅外部 API）甚至可以**唔納入 readiness**，只打 metrics／告警，避免一個次要依賴拖垮整組就緒。

**健康檢查超時／性能**

- Indicator 入面做慢查詢／遠程呼叫 → 探針超時 → K8s 當失敗；嚴重時仲會佔住 Tomcat／管理線程。
- 做法：探活用**短超時**（例如 1–2s）、連線池獨立、失敗快速返回 `DOWN`；可以背景刷新狀態（定時探一次、探針只讀快取結果），避免每次 probe 都打滿下游。
- 唔好喺 health 入面跑重業務邏輯或全表掃描。

**Management 暴露面**

- `management.endpoints.web.exposure.include`：**白名單**（例如只開 `health,info,prometheus,metrics`）。
- 危險端點：`env`、`heapdump`、`threaddump`、`shutdown`、`configprops` —— 生產預設唔暴露；要開就掛**獨立 management.port** + 內網／mTLS／閘道認證，唔好同業務端口混公開。
- `show-details`／`show-components`：對外只回整體 `UP`/`DOWN`；細節限授權角色（`when_authorized`）。
- Health response **唔好**帶連線字串、密碼、內部 hostname 清單。

### 追問／陷阱

- 「Health DOWN = 要重啟？」→ **唔一定**。Readiness DOWN 只係暫時唔收流量；Liveness DOWN 先至殺。分唔清就會用依賴抖動換重啟風暴。
- 「K8s 兩個探針都打同一個 `/actuator/health`」→ 等於 liveness 綁晒所有 indicator；Redis 一抖就殺 Pod。要拆 group 或分 path。
- 「開晒 Actuator 就監控齊」→ health 只係存活／就緒訊號；仲要 metrics、日誌、tracing。見 [優雅停機](graceful-shutdown.md)（停機時 readiness 先變唔健康再收流量）。
- 自訂 `HealthIndicator`：記得處理異常（catch 後回 `DOWN`），唔好令探針本身拋 500 拖垮管理端口。

## 關聯

- [Bean 生命週期](bean-lifecycle.md)
- [優雅停機](graceful-shutdown.md)
- [熔斷降級](../distributed/circuit-breaker-degradation.md)
