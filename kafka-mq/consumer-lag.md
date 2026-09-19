# Consumer Lag：怎麼算、為何飆、怎麼壓

## 面試問題

Kafka Consumer Lag 是什麼？怎麼算？線上 lag 一直漲，你會怎麼排查？

## 答法

**定義**

- Lag ≈ 該 partition **最新 log end offset** − 該 consumer group **已提交的 committed offset**。
- 表示「還有多少條還沒被這個 group 確認處理完」，不是「還在 JVM 記憶體裡的條數」。

**怎麼看**

- `kafka-consumer-groups.sh --describe`、監控系統的 `records-lag` / max lag。
- 看 **per-partition**，不要只看 group 總和（可能某一 partition 卡住、其他正常）。

**常見原因（面試愛問）**

1. **消費慢**：單條處理重（DB、RPC、大 JSON）；batch 太大或太小。
2. **卡住**：下游超時、死鎖、線程池滿、某條 poison message 反覆失敗。
3. **分配不均**：key 熱點 → 某一 partition lag 獨高。
4. **rebalance 頻繁**：成員進出、`session.timeout` / `max.poll.interval` 配錯 → 反覆停消費。
5. **生產突增**：流量尖峰，消費吞吐跟不上。

**怎麼壓（對症）**

- 加 consumer 實例（partition 數夠才有用）；或先加 partition 再擴 consumer。
- 業務異步化 / 批量寫庫；失敗進死信，別堵主循環。
- 熱點 key 打散；調 `max.poll.records`、下游超時與併發。
- 先確認是「真處理慢」還是「沒 commit」——後者 lag 數字也會騙人。

## 常見追問 / 陷阱

「Lag = 0 就代表剛好一次、沒丟消息？」

→ 否。Lag 只反映 **committed offset 跟上 tip 的距離**。先 commit 再處理可能 **丟**；至少一次仍可能 **重複**。業務冪等跟 lag 監控是兩件事。

## 小練習

某 partition lag 十萬、其他幾乎為 0——你第一個懷疑什麼？會查哪三個指標？
