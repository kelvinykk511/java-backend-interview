# Kafka：冪等 Producer 與 Exactly-Once

## 面試問題

Kafka 的「冪等 Producer」解決什麼？跟「Exactly-Once（EOS）」差在哪？後端寫入下游 DB 時還夠不夠？

## 答法

- **至少一次（at-least-once）**：broker 確認前網路超時，Producer 重試 → 可能重複訊息。
- **冪等 Producer**（`enable.idempotence=true`）：同一個 Producer 對**同一個 partition** 帶 `PID + sequence`；broker 丟棄重複序号 → **單分區、單會話內不重複寫 log**。
- **Exactly-Once（事務 Producer）**：跨多 partition／「讀-處理-寫」時用 transactional id；搭配 `isolation.level=read_committed` 的 consumer，才接近端到端 EOS（Kafka 內部）。
- **對 DB 不夠**：Kafka 不重複 ≠ 你的消費端寫 MySQL 不重複。消費端仍要：**業務冪等鍵**（訂單號）、唯一索引、或「先寫 outbox／再標 offset」等。
- **面試一句話**：冪等 Producer 防「重試造成的重複寫入同一 partition」；真正業務 exactly-once 還要消費端冪等 +（必要時）Kafka 事務。

## 常見追問 / 陷阱

「開了冪等就永遠不重複？」

→ 不是。Producer 重啟換 PID、換 partition、或下游 DB 重試，仍可能重複。面試官常接著問：消費端用訂單號做唯一約束怎麼設計。

## 小練習

充值回調先寫 Kafka、再由消費者入帳。只開冪等 Producer、消費者用「自動 commit」且入帳 SQL 無唯一約束——最容易在哪一步重複入帳？你會改哪兩處？
