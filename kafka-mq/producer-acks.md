# Kafka Producer acks

## 面試問題

Kafka Producer 的 `acks` 怎麼配？和可靠性、吞吐怎麼權衡？

## 答法

`acks` 決定「寫成功」要等多少副本確認：

| 值 | 行為 | 取捨 |
|----|------|------|
| `0` | 發出就當成功，不等 broker | 最快，可能丟 |
| `1` | Leader 寫入本地 log 即 OK | 常見折中；Leader 掛掉且未複製時可能丟 |
| `all` / `-1` | ISR 裡所有副本都確認 | 最穩；慢一點，需配合 `min.insync.replicas` |

實務交易/資金類：多半 `acks=all` + 合理重試 + 冪等 producer（`enable.idempotence`），下游仍按至少一次做冪等。

別只盯 acks：還有 `retries`、`linger.ms` / batch、超時。acks 高但重試亂配，仍可能重複寫。

## 常見追問 / 陷阱

「`acks=all` 就絕對不丟？」

→ 不保證。若 ISR 縮到只剩 Leader，而 `min.insync.replicas` 又設太低，仍可能在故障窗口丟。要一起看 ISR 與 `min.insync.replicas`。
