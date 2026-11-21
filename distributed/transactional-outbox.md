# Transactional Outbox（事務性發件箱）與業務冪等

## 面試問題

本地 DB 寫成功後要發 Kafka，怎麼保證「庫裡有這筆、消息也一定會出去」，又不會雙寫不一致？和業務冪等怎麼搭配？

## 答法

**問題本質**

- 先寫庫再發 MQ：庫成功、MQ 失敗 → 下游永遠收不到。
- 先發 MQ 再寫庫：MQ 成功、庫失敗 → 下游處理了幽靈事件。
- 兩邊各有自己的事務，**沒有跨庫 XA 的簡潔銀彈**（實務很少靠 2PC）。

**Outbox 做法（面試主線）**

1. **同一本地事務**：業務表 + `outbox` 表一起 `INSERT`（或狀態更新 + outbox 行）。
2. **提交後**：獨立 Relay（輪詢 / CDC 如 Debezium）把 outbox 行發到 Kafka。
3. **發送成功**：標記 outbox 已發送（或刪除）；失敗則重試 → 至少一次投遞。
4. **下游**：靠**業務冪等鍵**消化重複消息（見 [冪等](idempotency.md)）。

面試口訣：**本地事務綁定業務與 outbox；異步可靠投遞；消費者冪等**。

**和業務冪等怎麼拼（CEX 常考）**

| 環節 | 保證 | 典型手段 |
|------|------|----------|
| 寫庫 + outbox | 原子「有業務 ⇒ 有待發事件」 | 同一本地 TX |
| Relay → Kafka | at-least-once | 重試、CDC、發完標記 |
| 消費者 | 副作用只生效一次 | 業務唯一鍵 + 去重表／狀態機 |
| 對外 API | 客戶端重試不雙花 | `Idempotency-Key`／業務 requestId |

口訣：**Outbox 解決「會不會發出」；冪等解決「發多次／收多次會不會錯帳」**——兩邊缺一不可。

**CEX 例子**：充值入帳寫 `deposit` + outbox（`depositId`、事件類型、payload）；Relay 推「入帳完成」；風控／資產服務用 `depositId` 冪等，重複消息直接 ACK。

**和「先寫庫再發」比**：多一張表 + Relay，換來「庫成功 ⇒ 消息最終會出」的可證明保證。

## 常見追問／陷阱

「Outbox 算 Exactly-Once 嗎？」

→ 對生產端是「本地提交與待發事件原子」；對整條鏈仍是 **at-least-once + 下游冪等**。Kafka EOS（idempotent producer / txn）解決的是 broker 側重複，**替不了**業務 outbox。

另一陷阱：Relay 只掃 `status=NEW` 卻沒索引／沒批次上限 → 表變大後拖垮主庫；實務常 CDC 或分表歸檔。

再一個：outbox 的「業務鍵」和 API 的 `Idempotency-Key` 不是同一個東西——前者綁領域實體（訂單號、`depositId`），後者綁「這一次客戶端請求」；提現場景兩者常要**同時**有（見冪等筆記小練習）。

## 小練習

充值入帳寫 `deposit` 成功，要通知風控服務。畫一下 outbox 一列至少要有哪些欄位（業務鍵、payload、狀態、重試次數）？消費者用哪個鍵做冪等？

## 關聯

- [介面與消息的冪等](idempotency.md)
- [Spring @Transactional 失效](../spring/transactional-failure-scenarios.md)
- [Kafka 消息順序](../kafka-mq/message-ordering.md)

---

## 深入（2026-11-21）：Relay 輪詢 vs CDC、標記時機、同事務陷阱

### 面試可以點追

「`@TransactionalEventListener(AFTER_COMMIT)` 喺 commit 後 `kafkaTemplate.send`，係咪已經夠？同 Outbox 差喺邊？Relay 用輪詢定 Debezium？發完先更新 outbox 定先標 SENT？」

### 答法

**AFTER_COMMIT 發 MQ ≠ Outbox**

- `AFTER_COMMIT`／`TransactionSynchronization.afterCommit` 解決「事務未提交就發、下游查唔到」同「rollback 咗但消息已飛」。
- **救唔到**：「commit 成功 → 進程 crash／Kafka 短暫不可用 → 呢次 send 永遠唔會再試」。進程重啟後冇「待發清單」。
- Outbox：待發事件**寫喺庫裡**，同業務同一事務提交；Relay 可以 crash 重試，最終會出。

**輪詢 vs CDC**

| | 輪詢 outbox 表 | CDC（binlog → Debezium 等） |
|---|---|---|
| 優點 | 實作簡單、可控批次／退避 | 近實時、少打主庫業務表、天然跟事務提交順序 |
| 代價 | 週期掃表、要複合索引（`status, id`）、表膨脹要歸檔 | 運維／權限／schema 演進；要處理 snapshot、DDL |
| CEX 常見 | 中小流量、自建 Relay | 入帳／劃轉等要低延遲、已有 Canal／Debezium |

兩邊都係 **at-least-once**：Relay 發成功但標記失敗會重發 → 下游必須冪等。

**標記時機（容易講錯）**

1. **讀 NEW → send → 標 SENT**（最常見）：send 成功、標 SENT 失敗 → 重發（重複 OK，靠冪等）。
2. **先標 SENDING 再 send**：避免多個 Relay 搶同一行；要有租約／`locked_by` + 超時回收，否則卡死。
3. **刪除行**：可以，但排障冇歷史；多數留 SENT + 定期歸檔。
4. **唔好**「先標 SENT 再 send」：標完 crash、未 send → **丟消息**。

**同事務／傳播陷阱（同昨日 Kafka 順序、雙寫一齊考）**

- 業務 + outbox 必須同一 `REQUIRED` 事務；**唔好**對 outbox 單獨 `REQUIRES_NEW` 而業務回滾 → 幽靈事件。
- 事務入面直接 `kafkaTemplate.send`（無 outbox）：rollback 後消息可能已出；commit 前發出下游讀未提交。
- Relay **唔好**同業務請求共用 Tomcat 線程同步狂掃；獨立進程／線程池，失敗有退避同死信指標。

**表設計最少欄位（面試可講）**

`id`, `aggregate_type`, `aggregate_id`（如 `depositId`）, `event_type`, `payload`（JSON）, `status`（NEW/SENT/DEAD）, `created_at`, `sent_at`, `retry_count`；可選 `trace_id`。唯一約束可放 `(aggregate_id, event_type)` 防業務重複插入。

### 追問／陷阱

- 「開咗 Kafka 冪等 producer／EOS 就唔使 Outbox？」→ 否。EOS 管 broker 側重複同讀未提交；**管唔到**「DB 成功但應用根本冇再 send」。
- 「只用 Canal 訂業務表、唔寫 outbox？」→ 可以當「隱式 outbox」，但要把「可發佈事件」同表變更語義對齊（唔好把中間狀態當最終事件），並處理 schema／多表拼裝。
- Relay 同 key 亂序：Outbox 按 `id` 遞增發，通常近似提交序；跨分區仍無序，下游要版本號／狀態機（見 [消息順序](../kafka-mq/message-ordering.md)）。
