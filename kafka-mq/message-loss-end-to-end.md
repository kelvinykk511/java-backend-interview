# Kafka 消息丟失全鏈路排查

## 面試問題

「Kafka 點樣會丟消息？由 producer 到 consumer 逐段講，同埋點樣配置先唔丟？」

## 答法（分三段講）

### 1. Producer 段（發出去但冇真正寫入）
- **`acks=0`** 唔等確認；**`acks=1`** leader 寫完就回，leader 未同步 follower 就死 → 新 leader 冇呢條 → 丟。
- 用 **`acks=all` + `enable.idempotence=true`**（Kafka 3.0 起 producer 預設開 idempotence，即 acks=all、retries 大、`max.in.flight.requests.per.connection<=5` 保序）。
- **`send()` 係異步**：fire-and-forget 唔睇 callback / `Future`，錯誤被吞 → 業務以為發咗。必須處理 callback 異常。
- **`delivery.timeout.ms`** 到期仍失敗會拋異常；要落本地重試表 / outbox，唔可以只 log。
- 消息仲喺 producer 內存 buffer（**RecordAccumulator**）時進程死 → 丟；關鍵業務用 **Transactional Outbox**。
- 進程退出前 `producer.flush()` / `close()`，否則 buffer 入面嘅 batch 會冇咗。

### 2. Broker 段（寫入咗但副本唔夠）
- **`replication.factor=3` + `min.insync.replicas=2`**，配合 `acks=all`：ISR 少過 2 時寫入直接報 **NotEnoughReplicas**，寧願寫失敗都唔偷偷丟。
- **`unclean.leader.election.enable=false`**（0.11 起預設 false）：唔俾落後嘅非 ISR 副本做 leader。
- Kafka 主要靠 **副本** 而唔係每條 fsync 保證持久（寫 page cache）；同一機櫃一齊斷電可以丟 → **`broker.rack`** 機架感知分散副本。

### 3. Consumer 段（攞到但冇處理完）
- **`enable.auto.commit=true`** 按時間自動提交 offset，唔理你處理完未 → 提交後處理前 crash = 丟。
- poll 完丟入線程池異步處理再即刻 commit，一樣丟。
- 正解：**處理完先手動 commit**（`commitSync` / 批量 commit）→ at-least-once；配合消費端 **冪等**（業務唯一鍵 / 去重表）處理重複。
- Rebalance：喺 `onPartitionsRevoked` 提交已處理嘅 offset，避免重複或漏。

## 一句總結

> Producer `acks=all` + idempotence + 睇 callback；Broker RF=3 + `min.insync.replicas=2` + 禁 unclean；Consumer 處理完先 commit + 冪等。唔丟嘅代價係 at-least-once，重複要靠冪等。

## 常見追問 / 陷阱

- **`acks=all` 但 `min.insync.replicas=1`**（broker 預設值就係 1）：ISR 縮到得 leader 時，`acks=all` 實際等於 `acks=1`，照丟。
- **consumer lag 大過 `retention.ms` / `retention.bytes`**：未消費就被刪，表面似「丟消息」，要監控 lag。
- **唔丟 ≠ 唔重複**：端到端 exactly-once 只限 Kafka→Kafka 嘅 transactional read-process-write；一般業務靠 consumer 冪等。
- 「開咗 `enable.idempotence` 就 exactly-once？」→ 只係單 Producer 單分區唔重複，端到端仲要 consumer 冪等或 Kafka 事務。
- 交易所場景：充值/成交消息丟咗要靠 **對帳**（按 offset/序號檢查斷號，定時同 DB 對）兜底，唔可以只信 MQ 配置。

相關：[producer-acks](producer-acks.md)、[isr-min-insync](isr-min-insync.md)、[consumer-group-offset-delivery](consumer-group-offset-delivery.md)、[transactional-outbox](../distributed/transactional-outbox.md)
