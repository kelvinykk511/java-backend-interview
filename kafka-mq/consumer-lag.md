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


---

## 深入（2026-11-11）：lag 持續漲 + 頻繁 rebalance 點排查

### 面試問題

線上某個消費組 lag 持續上漲，監控同時見到頻繁 rebalance。你點分「真係處理唔切」同「被自己踢出組」？`session.timeout.ms`、`max.poll.interval.ms`、`group.instance.id` 各解什麼問題？亂加大 poll interval 有什麼代價？

### 答法

**先分流，唔好一開始就加 partition／加實例**

- Lag = tip − committed（按 **partition** 睇，唔好只看 group 總和）。
- 同時睇：**rebalance 次數**、成員 join／leave、係邊個 member 被踢、`records-lag` 係全部 partition 一齊升定得一兩條卡死。
- 兩類主因：
  1. **吞吐真跟唔上**：處理慢、熱點 key、並行度不夠（有效 consumer ≤ partition）。呢種即使冇 rebalance，lag 都會斜向上。
  2. **Rebalance 循環**：消費窗口不停停低，lag 被「停消費」推高，再觸發更多超時踢人。

**兩個超時唔好混（0.10.1 之後 heartbeat 喺獨立線程）**

| 參數 | 睇什麼 | 超時後果 |
|------|--------|----------|
| `session.timeout.ms`（配 `heartbeat.interval.ms`） | 進程仲唔喺度（心跳） | 當死機／斷網踢走 |
| `max.poll.interval.ms` | 兩次 `poll()` 之間最長間隔 | 活住但批次處理太久／堵咗 poll 循環 → 一樣被踢 |

面試常考：GC／下游慢／`max.poll.records` 太大，心跳其實仲喺跳，但 poll 間隔爆咗 → 日誌係被踢出組，唔係 session 心跳失敗。

**同 rebalance 的惡性循環**

- 被踢 → 觸發 rebalance（Eager 會撤銷全部；Cooperative 只撤必須搬走的，見 [consumer-rebalance](consumer-rebalance.md)）→ 窗口內冇人消費 → lag 衝高。
- 滾動發佈冇靜態成員：每次下線都當「成員離開」。
- 處理越慢越容易爆 `max.poll.interval` → 越 rebalance 越慢。

**對症，而唔係只調大超時**

- 縮細 `max.poll.records`、加快處理、毒消息進死信，**保證 poll 循環唔被堵**。
- 滾動發佈：`group.instance.id`（靜態成員）+ cooperative sticky；協調器會等，唔即刻當新人。
- 並行度：consumer 數 ≤ partition 數；熱點先打散 key。
- **亂加大 `max.poll.interval.ms`**：短期 rebalance 停、lag 看似穩，但卡住／死鎖／毒消息嘅 member **更耐先被踢**，對應 partition 可以靜默落後好耐。呢個係常見假好轉。

**再加一個語意陷阱**

- 指標 lag 睇的是 **committed**。若先 commit 再異步處理，lag 可以係 0，但業務其實仲有 in-flight；進程掛了可能丟。Lag 低 ≠ 業務追上，更 ≠ 冇重複。

### 追問／陷阱

- 「加多幾倍 `max.poll.interval.ms` 就搞得掂？」→ 只係延後踢人；根因係批次太大或 poll 被堵。卡住嘅分區會更遲先暴露。
- 「lag 飆即係消息丟了？」→ 唔係；消息仲喺 log（未超 retention）。丟／重複睇 commit 時機同冪等。
- 只擴 consumer 但 partition 少過實例：多餘實例空轉，幫唔到 lag。
