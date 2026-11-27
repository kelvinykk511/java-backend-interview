# Consumer Group、Offset 與至少一次投遞

## 面試問題

Kafka 裡 Consumer Group 和 Offset 是什麼？為什麼常說「至少一次」而不是「剛好一次」？

## 答法

**Consumer Group**

- 同一 Group 內，一條 partition 同一時間只給一個 consumer 消費 → 水平擴展、同一 topic 可被多個業務各自用不同 group 訂閱。
- 再平衡（rebalance）：成員增減或 partition 數變時，會重新分配；期間可能短暫停消費或重複消費。

**Offset**

- 每個 group 對每個 partition 記「讀到哪」。
- 提交時機：自動（可能丟/重複）或手動；業務處理成功後再 commit 更穩。

**投遞語義（常考）**

| 語義 | 意思 | 典型做法 |
|------|------|----------|
| 最多一次 | 可能丟、不重複 | 先 commit 再處理 |
| **至少一次** | 不丟、可能重複 | 先處理再 commit（最常見） |
| 剛好一次 | 不丟不重複 | Kafka 事務 / 冪等 producer + 下游冪等，成本高 |

實務預設按「至少一次」設計，下游自己做冪等。

## 常見追問 / 陷阱

「commit 了就一定不會再收到這條？」

→ 不一定。Rebalance、重複投遞、手動錯位 commit，都可能讓同一 offset 區間再被消費。要靠業務冪等兜底。

## 小練習

為什麼「先處理再 commit」會重複，但「先 commit 再處理」可能丟消息？

## 關聯

- [Consumer Rebalance](consumer-rebalance.md)
- [Consumer Lag](consumer-lag.md)
- [剛好一次消費](exactly-once-consumer.md)
- [介面與消息的冪等](../distributed/idempotency.md)


## 深入：提交時機、batch 空洞、併發處理（2026-11-27 warm-up）

**面試題**：入帳 consumer 同一個 group。先用 `enable.auto.commit=true`，poll 完一批、逐條入帳，全部做完先下一次 poll。之後改手動：batch 入面第 3 條失敗，仍然 `commitSync` 咗呢批最後一條嘅 offset；又或者丟去線程池併發處理，提交「見到嘅最大 offset」。發版 rebalance 時冇喺 `onPartitionsRevoked` 提交。結果有啲充值重複入帳，有啲永遠冇入但 offset 已經超前。點解釋、點改？

- **Offset 記嘅係「下一條要讀」**。Java API 無參嘅 `commitSync()`／自動提交會提交 consumer 嘅 position（已 poll 嘅最後 offset + 1）。自己組 map 時如果提交 `record.offset()` 而唔係 `offset() + 1`，最後一條會被再消費一次。
- **先處理再提交 = 至少一次**。成功入帳之後先 commit；崩潰或 rebalance 發生喺 commit 之前，就會重複消費 → 下游必須冪等。**先 commit 再處理** 可能丟：commit 完、入帳前進程掛咗，呢組唔會再投遞給同一個 group。
- **`enable.auto.commit=true`**：下一次 `poll()` 時（且過咗 `auto.commit.interval.ms`）先提交**上一批**已返回嘅 offset。單線程「處理完先再 poll」通常係至少一次，唔係剛好一次。陷阱係**未處理完就再 poll**——最常見係把 record 丟去線程池之後主線程即刻 poll，自動提交就等於先 commit 再處理，失敗或崩潰會丟消息。
- **Batch 中間失敗仍然提交最後 offset = 丟消息**。第 3 條失敗、卻提交咗第 50 條，等于告訴 broker「到 50 都完成」。同 group 再平衡之後唔會再派第 3 條。要麼唔提交、等重試；要麼只提交「連續成功前綴」（第 1、2 條），失敗條同後面先留低。
- **併發處理唔可以提交「見到嘅最大 offset」**。1–4 並行，4 先完成就 commit 4，1–3 若失敗或未完成就出現**空洞**：offset 超前、中間永遠唔會再被呢個 group 拉到。只能提交最高**連續已完成**嘅 offset；更簡單係同一 partition 保持單線程順序處理。
- **Rebalance**：partition 被收回前要喺 `onPartitionsRevoked` 用 `commitSync` 提交已完成嘅 offset，否則新 owner 從舊 committed 位置重放（重複）。`commitAsync` 唔保證成功、唔會自動重試，循環入面可以用 async，但 close／revoke 必須 sync。
- **同 group 同一條 partition 同一時間只有一個 consumer**。水平擴展靠加 partition，唔係多條線程共享同一個 `KafkaConsumer`（佢唔線程安全）。

### 追問

- 「commit 咗就一定唔會再收到？」唔一定。revoke 前未提交、手動提交錯位、至少一次重放，都會再嚟。靠業務冪等兜底。
- 自動提交唔等於最多一次，亦唔等於剛好一次。
- Kafka 事務／`read_committed` 先同「剛好一次」有關，成本高；入帳實務多數係至少一次 + 以 `depositId`／`txHash` 冪等。
