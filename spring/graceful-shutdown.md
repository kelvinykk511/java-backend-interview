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
