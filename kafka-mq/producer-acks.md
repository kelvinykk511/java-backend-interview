# Kafka Producer acks

## 面試問題

Kafka Producer 的 `acks` 怎麼配？和可靠性、吞吐怎麼權衡？還要和哪個 broker 參數一起看？

## 答法

`acks` 決定「寫成功」要等多少副本確認：

| 值 | 行為 | 取捨 |
|----|------|------|
| `0` | 發出就當成功，不等 broker | 最快，可能丟 |
| `1` | Leader 寫入本地 log 即 OK | 常見折中；Leader 掛掉且未複製時可能丟 |
| `all` / `-1` | ISR 裡所有副本都確認 | 最穩；慢一點，需配合 `min.insync.replicas` |

**實務口訣**

- 交易／資金類：多半 `acks=all` + 合理重試 + 冪等 producer（`enable.idempotence`），下游仍按至少一次做冪等。
- `acks=all` 真正意義是「ISR 全員」；ISR 縮小時行為靠 `min.insync.replicas` 卡下限（不夠就拒寫，寧願失敗也不假成功）。
- 別只盯 acks：還有 `retries`、`linger.ms`／batch、超時。acks 高但重試亂配，仍可能重複寫（所以要冪等／業務鍵）。

## 常見追問 / 陷阱

「`acks=all` 就絕對不丟？」

→ 不保證。若 ISR 縮到只剩 Leader，而 `min.insync.replicas=1`，故障窗口仍可能丟。要一起看 ISR 與 `min.insync.replicas`（常見 ≥2）。

另一陷阱：把 `acks=all` 當 Exactly-Once——它只是「寫入路徑更不易丟」；端到端還要冪等／事務／下游去重。

## 小練習

設了 `acks=all` 但 `min.insync.replicas=1`，跟 `acks=1` 在「Leader 獨活」時差在哪？一句話。
