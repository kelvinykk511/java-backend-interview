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

---

## 深入（2026-11-02）：Kafka 事務 + consume-transform-produce

**面試問題**：「讀 topic A → 處理 → 寫 topic B」點樣做到 exactly-once？Kafka 事務點樣實現？

**答法**

- 冪等 producer（PID + 每分區 sequence）只保證**單分區、單會話**唔重複；跨分區、重啟後就唔夠 → 要事務。
- 設 `transactional.id`（穩定、每個實例唯一，例如按輸入分區分配）→ 自動開冪等。
- 流程：`initTransactions()` → `beginTransaction()` → `send()` 去 B → `sendOffsetsToTransaction(offsets, consumer.groupMetadata())` → `commitTransaction()`。
  - 重點：**消費 offset 同輸出消息喺同一個事務提交**，要麼一齊成功要麼一齊作廢。consumer 要關 `enable.auto.commit`。
- 背後：Transaction Coordinator 寫 `__transaction_state`，兩階段提交，最後喺每個分區寫 **commit/abort marker**。
- 下游 consumer 要 `isolation.level=read_committed`（預設係 `read_uncommitted`！），只讀到 **LSO**（Last Stable Offset）之前嘅數據，abort 咗嘅消息會被過濾。

**追問／陷阱**

- **殭屍實例（zombie fencing）**：舊實例 GC 停頓咗，新實例用同一個 `transactional.id` 啟動，`initTransactions()` 會令 epoch +1，舊實例再寫就收 `ProducerFencedException` → 必須關閉，唔好 retry。
- 一個未提交嘅長事務會卡住 LSO → read_committed 嘅 consumer lag 睇落飆升但其實係等緊事務（`transaction.timeout.ms` 預設 60s）。
- EOS 只喺 **Kafka 到 Kafka** 之內成立。寫 DB、調 HTTP、發郵件呢啲外部副作用唔會跟住 rollback → 外部仍然要冪等鍵 / outbox。
- 有成本：每次 commit 要寫 marker，事務太細吞吐跌，通常按批（例如每 100ms 或每 N 條）一個事務。
