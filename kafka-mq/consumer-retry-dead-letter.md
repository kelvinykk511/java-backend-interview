# Kafka 消費失敗：重試 + 死信（DLT）設計

> 相關：[消息順序](./message-ordering.md)、[consumer rebalance](./consumer-rebalance.md)、[冪等](../distributed/idempotency.md)、[消息丟失](./message-loss-end-to-end.md)

## 面試問題

消費者處理某條消息失敗（例如下游返 503），點設計重試同死信，**唔好卡死成個 partition，又唔丟消息**？

## 模範答案

- **先分類錯誤**：
  - 可重試：超時、503、鎖衝突。
  - 唔可重試（毒消息）：反序列化失敗、數據非法、業務規則拒絕 → 唔好重試，直接入 DLT。
- **第一層：原地短重試**：少次數 + 退避（Spring Kafka `DefaultErrorHandler` + `ExponentialBackOff`，失敗會 seek 返去同一 offset）。總等待時間要遠小於 `max.poll.interval.ms`，否則被踢出 group 觸發 rebalance。
- **第二層：重試 topic（非阻塞）**：例如 `orders-retry-10s` → `orders-retry-1m` → `orders-retry-10m` → `orders-dlt`。原 partition 提交 offset 繼續行，唔阻塞後面消息。Spring 用 `@RetryableTopic`。
- **DLT 要帶齊上下文**：headers 記原 topic/partition/offset、異常類型同 message、重試次數、首次失敗時間；有告警 + 人手/工具重放；重放必須冪等。
- **送 DLT 失敗就唔好提交 offset**，否則真係丟咗。
- **整體係 at-least-once** → 消費端必須冪等（業務唯一鍵 / 去重表）。

## 常見追問／陷阱

- **重試 topic 會打亂順序**：同一 key 嘅後續消息會比失敗嗰條先處理。對順序敏感（帳戶入帳/扣款）：
  - 要麼原地阻塞重試 + 告警（犧牲吞吐保順序）；
  - 要麼做 **key 級 park**：一條入咗 retry，同 key 後續消息都要跟住排去 retry/暫存，直到前面成功。
- **喺 retry consumer 入面 `Thread.sleep` 等延遲** → 超 `max.poll.interval.ms` → rebalance 風暴。正確做法係 `pause/resume` 分區（Spring 已內置 back-off 機制）。
- **反序列化錯誤發生喺 listener 之前**：唔配 `ErrorHandlingDeserializer` 會不斷拉同一條、無限循環。
- 唔好將所有異常都重試：業務校驗失敗重試 100 次都係失敗，只會塞爆 retry topic。
