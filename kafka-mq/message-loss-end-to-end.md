# Kafka 消息丟失全鏈路排查

## 面試問題

「Kafka 會唔會丟消息？如果要保證唔丟，由 Producer 到 Broker 到 Consumer 每一段你會點配置？」

## 答法（分三段講）

### 1. Producer 段（發出去但冇真正寫入）
- **`send()` 係異步**：唔睇 callback / `Future`，失敗咗都唔知 → 一定要處理 callback，失敗落庫或告警。
- `acks=all`（等 ISR 全部確認），`acks=0/1` 都有丟嘅窗口。
- `retries` 夠大 + `delivery.timeout.ms` 合理；開 `enable.idempotence=true` 防重試造成重複同亂序。
- 進程退出前 `producer.flush()` / `close()`，否則 buffer 入面嘅 batch 會冇咗。
- 業務關鍵消息：用 **Transactional Outbox**（先同業務數據同一個 DB 事務落 outbox 表，再異步發），避免「DB 提交咗但消息冇發」。

### 2. Broker 段（寫入咗但副本唔夠）
- `replication.factor >= 3`，`min.insync.replicas = 2`：ISR 少過 2 就拒寫，寧願報錯都唔假成功。
- `unclean.leader.election.enable=false`：唔俾落後嘅副本做 Leader，否則會截斷已確認嘅數據。
- Kafka 默認靠 **副本** 保證可靠，唔係靠每條 fsync（page cache）；所以副本數同 ISR 先係關鍵。

### 3. Consumer 段（攞到但冇處理完）
- **自動提交 offset**（`enable.auto.commit=true`）：offset 提交咗但業務仲未處理完就 crash → 丟。
- 正確做法：**先處理、後手動提交**（`commitSync` / 批量後提交）→ 變成「至少一次」，所以下游要做 **冪等**（業務唯一鍵 / 去重表）。
- 異步線程池處理：主線程 poll 完就提交，但線程池入面嘅任務未做完 → 都會丟；要等處理完先提交對應 offset。
- Rebalance：喺 `onPartitionsRevoked` 提交已處理嘅 offset，避免重複或漏。

## 一句總結

> Producer `acks=all` + 重試 + 冪等 + 處理 callback；Broker 3 副本 + `min.insync.replicas=2` + 禁 unclean 選舉；Consumer 手動提交、處理完先提交 + 下游冪等。唔丟嘅代價就係「可能重複」，所以冪等係必答。

## 常見追問 / 陷阱

- 「`acks=all` 就一定唔丟？」→ 唔係。如果 `min.insync.replicas=1`，ISR 縮到得 Leader 時，`all` 等於 `1`。
- 「開咗 `enable.idempotence` 就 exactly-once？」→ 只係單 Producer 單分區唔重複，端到端仲要 consumer 冪等或 Kafka 事務（read-process-write）。
- 交易所場景：充值/成交消息丟咗要靠 **對帳**（按 offset/序號檢查斷號，定時同 DB 對）兜底，唔可以只信 MQ 配置。

相關：[producer-acks](producer-acks.md)、[isr-min-insync](isr-min-insync.md)、[consumer-group-offset-delivery](consumer-group-offset-delivery.md)、[transactional-outbox](../distributed/transactional-outbox.md)
