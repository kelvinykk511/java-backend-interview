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

## 關聯

- [Bean 生命週期](bean-lifecycle.md)
