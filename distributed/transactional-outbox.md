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
