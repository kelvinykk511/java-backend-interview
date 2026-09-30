# Consumer Lag：怎麼算、為何飆、怎麼壓

## 面試問題

Kafka Consumer Lag 是什麼？怎麼算？線上 lag 一直漲，你會怎麼排查與緩解？

## 答法

**定義**

- 每個 partition：`lag ≈ log end offset（high watermark／LEO）− 該 group 的 committed offset（或 consumer 當前 position，視你看的指標）`。
- 語意：「這個 group 在該 partition 上還落後 tip 多少」——**不是**「消息丟了」，也不是 JVM 裡堆積的條數。

**怎麼觀察**

- `kafka-consumer-groups.sh --describe`：看 CURRENT-OFFSET、LOG-END-OFFSET、LAG（**按 partition**）。
- 監控：`records-lag` / `records-lag-max`；盯 per-partition，別只看 group 總和（可能單 partition 卡住）。

**常見原因**

1. **處理慢**：單條重（DB／RPC）、批次不當；STW GC 拖長 poll 間隔。
2. **Rebalance 抖動**：成員進出、`session.timeout` / `max.poll.interval.ms` 配錯 → 反覆停消費。
3. **並行度不夠**：consumer 數 > partition 數也沒用；常是 **partition < 有效消費者** 或熱點 key 打到少數 partition。
4. **卡住**：下游超時、死鎖、poison message 反覆失敗、線程池滿。
5. **生產突增**：吞吐跟不上。

**緩解（對症）**

- **水平擴 consumer**（上限 ≈ partition 數）；不夠就先加 partition 再擴。
- 異步／批量處理時想清楚 **commit 語意**（先處理再 commit vs 先 commit 可能丟）。
- 調大 `max.poll.interval.ms` **只有**在理解風險時才做：間隔太長 → 卡住的 member 更晚被踢，rebalance／卡死更久。
- 熱點 partition：打散 key、業務削峰；失敗進死信，別堵 poll 循環。
- `max.poll.records` 加大前先確認單次批次處理能力，否則更容易觸發 max.poll.interval → 被踢出組。

**面試一句話**：Lag = tip − committed；先分清「真慢／沒 commit／rebalance／熱點」，再擴並行或改處理，別把 lag 當丟消息。

## 常見追問 / 陷阱

「Lag ≠ 丟消息？」

→ 對。Lag 只反映 **committed（或 position）離 tip 的距離**。消息仍在 log 裡（未超 retention）。丟／重複取決於 **何時 commit** 與業務冪等。

「committed offset vs in-flight」

→ 已 commit 之前的 in-flight 若進程掛了會重投（at-least-once）。Lag 看 committed 時，可能低估「手頭還沒做完」的量。

「狂加 `max.poll.records`」

→ 單次 poll 太多 → 處理超時 → 超過 `max.poll.interval` → 被踢 → rebalance → lag 更糟。容量沒跟上別只調大。

## 小練習

某 partition lag 十萬、其他幾乎為 0——你第一個懷疑什麼？會查哪三個指標？若把 `max.poll.interval.ms` 調到很大，短期 lag 看似穩了，長期可能藏什麼問題？
