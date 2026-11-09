# Spring Boot 優雅停機（Graceful Shutdown）

## 面試問題

K8s 滾動發佈時，舊 Pod 被殺，偶爾出現幾個 502 / 請求中斷 / MQ 消息重複。Spring Boot 怎樣做到「優雅停機」？光開 `server.shutdown=graceful` 夠不夠？

## 答法

**停機順序（要講清楚誰先誰後）**

1. K8s 發 **SIGTERM** → JVM 觸發 shutdown hook → Spring `ApplicationContext.close()`。
2. `server.shutdown=graceful`：Web 容器**停止接新請求**，等在途請求做完，最多等 `spring.lifecycle.timeout-per-shutdown-phase`（默認 30s）。
3. `SmartLifecycle` 按 phase 反序停止：Kafka listener container 停止拉取並提交 offset、調度器停止。
4. Bean 銷毀：`@PreDestroy` / `DisposableBean`，關線程池、連接池。
5. 超過 K8s `terminationGracePeriodSeconds`（默認 30s）→ **SIGKILL**，什麼都來不及。

**為什麼光開 graceful 不夠**

- **流量摘除有延遲**：SIGTERM 和「從 Service Endpoints 摘掉」是**並行**的，kube-proxy / 網關還可能把新請求打過來 → 502。解法：`preStop` 先 `sleep 5~10s`，讓摘流量先生效；readiness 探針在停機時轉為不健康（Boot 2.3+ 的 availability state）。
- **時間預算要對齊**：`preStop sleep + Spring 等待時間 < terminationGracePeriodSeconds`，否則被 SIGKILL 截斷。
- **自建線程池**：`ThreadPoolTaskExecutor` 要設 `setWaitForTasksToCompleteOnShutdown(true)` + `setAwaitTerminationSeconds`，否則任務直接丟；自己 `new` 的 `ExecutorService` Spring 不管，要自己在 `@PreDestroy` 裡 `shutdown()` + `awaitTermination()`。
- **MQ 消費**：停止前沒提交 offset → 重啟後重複消費 → 下游必須冪等（回到冪等那一課）。

## 常見追問 / 陷阱

「`kill -9` 能優雅停機嗎？」→ 不能，SIGKILL 不觸發 shutdown hook。陷阱二：容器裡 Java 不是 PID 1 的直接進程（例如 `sh -c java ...`），SIGTERM 被 shell 吃掉沒轉發 → 每次都等滿 30s 被 SIGKILL。解法：`exec java ...` 或用 tini/dumb-init。

## 小練習

出金服務 Pod 停機時，有一筆提現剛從 Kafka 拉到、正在調簽名機。描述從 SIGTERM 到進程退出，你希望這筆提現經歷什麼；如果被 SIGKILL 截斷，靠什麼保證不重複出金？

---

## 深入（2026-11-09）：時間預算、摘流量競態、自建資源

### 面試問題

K8s 滾動發佈時開咗 `server.shutdown=graceful`，仍然偶發 502／長請求被截斷／Kafka 重複消費。時間預算點對齊？SIGTERM 同 Endpoints 摘除點解會「搶跑」？自建線程池同 MQ listener 要補咩？

### 答法

**時間預算（一定要會算）**

| 旋鈕 | 作用 | 常見坑 |
|------|------|--------|
| `preStop` sleep（例如 5–15s） | 等 kube-proxy／網關把 Pod 從 Endpoints 摘走 | 唔 sleep → SIGTERM 當下仲有新請求打入 |
| `spring.lifecycle.timeout-per-shutdown-phase`（預設 30s） | Web／SmartLifecycle 等在途完成嘅上限 | 設太長會撞上 K8s 殺進程 |
| `terminationGracePeriodSeconds`（預設 30s） | SIGTERM → SIGKILL 總時限 | **必須 > preStop + Spring 等待**，否則中途被 SIGKILL |

口訣：`preStop sleep + Spring graceful 等待 < terminationGracePeriodSeconds`，仲要留幾秒緩衝俾 JVM 收尾。

例：`preStop=15s`、`lifecycle=30s`、`terminationGracePeriod=30s` → 15+30 > 30，長請求同 MQ 收尾一定被 SIGKILL 截斷。改法：`terminationGracePeriod=60`（或縮短 lifecycle／把長任務外置）。

**摘流量競態（502 主因）**

- K8s：**發 SIGTERM** 同 **從 Service Endpoints 移除** 係並行，唔保證「先摘流量再停機」。
- 結果：優雅停機已停收新請求（或進程開始拆），LB／kube-proxy 仲路由過嚟 → **502／connection reset**。
- 解法組合：
  1. `preStop` 先 `sleep`（或先打 readiness fail），等摘流量生效。
  2. Boot 2.3+：停機時把 readiness／availability 標成不可用，探針先失敗再等 SIGTERM 收尾。
  3. 網關／LB 連接排空時間要納入預算。

**自建資源同 MQ（光開 graceful 唔夠）**

- `ThreadPoolTaskExecutor`：`setWaitForTasksToCompleteOnShutdown(true)` + `setAwaitTerminationSeconds(...)`；自己 `new` 嘅 `ExecutorService` Spring **唔會**自動等，要喺 `@PreDestroy` `shutdown()` + `awaitTermination`。
- Kafka／MQ listener：靠 `SmartLifecycle` 停拉取；停之前要容許在途消息做完並 **commit offset**。被 SIGKILL 截斷 → 未提交 → 重平衡後**至少一次**再投遞 → 下游必須冪等。
- 長耗時出金／簽名：停機唔好假設「一定做完」；用狀態機 + 幂等鍵／outbox，重啟後可續跑或可安全重試。

**進程模型**

- 容器入口要用 `exec java ...`（或 tini），確保 Java 係收 SIGTERM 嘅主進程；`sh -c java ...` 常令 shell 食咗信號 → 每次都等滿 grace period 被 SIGKILL，「優雅」形同虛設。

### 追問／陷阱

- 「開咗 `server.shutdown=graceful` 就唔會 502？」→ 唔夠；要處理摘流量競態同時間預算。
- `kill -9`／SIGKILL：**唔會**跑 shutdown hook，任何優雅邏輯都無效。
- 把 `terminationGracePeriodSeconds` 同 `lifecycle` 都設 30、再加 `preStop 15` → 數學上必截斷，係發佈事故高發組合。

### 關聯

- [Actuator Health（liveness／readiness）](actuator-health.md)
