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
