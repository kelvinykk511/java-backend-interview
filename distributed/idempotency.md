# 介面與消息的冪等

## 面試問題

什麼叫冪等？HTTP 寫接口和 Kafka 消費者分別怎麼保證「同一業務請求來兩次，副作用只發生一次」？

## 答法

**定義（一句話）**

同一操作執行一次和執行多次，對系統可見狀態的影響一樣。

**為什麼需要**

- 客戶端重試、網關超時重放、Kafka **at-least-once** 都會讓同一業務進來多次。
- 交易所：充提回調、下單、劃轉——重複一次就可能錯帳。

**HTTP / API 側**

1. 客戶端帶 **`Idempotency-Key`（或業務 `requestId`）**，服務端以「用戶 + key」做去重鍵。
2. **DB 唯一約束**撞到 → 查已有結果原樣返回（不要再扣一次）。
3. 處理中可記 `PROCESSING`，重複請求返回「處理中」或等待，避免並行雙寫。

**MQ 消費側**

1. 消息帶業務唯一鍵（訂單號、`txHash`、outbox id）。
2. **先**本地冪等表 / 唯一索引，**再**做副作用；或狀態機 `PENDING → SUCCESS`，終態直接 ACK。
3. 和 Outbox 搭配：生產端 at-least-once，消費端必須冪等——整條鏈才安全。

**面試口訣**：**識別鍵 + 唯一約束/去重表 + 狀態機**；鎖只防併發，不替代「已完成」記錄。

## 常見追問 / 陷阱

「加了分布式鎖就冪等了嗎？」

→ 鎖只防**同時**進來的兩條。第一次已成功、鎖釋放後重試再進來，沒有「這筆已完成」記錄仍會再執行。

另一陷阱：只用「金額 + 地址」當充值去重鍵——同一地址同金額兩筆真實充值會被誤殺；應用鏈上 **`txHash`（或唯一 deposit id）**。

## 小練習

提現回調與用戶點「重試提交」可能撞在一起：`Idempotency-Key` 和鏈上 `txHash` 各解決哪一種重複？能不能只用其中一個？

## 關聯

- [Transactional Outbox](transactional-outbox.md)
- [Kafka Consumer Group／Offset](../kafka-mq/consumer-group-offset-delivery.md)


## 深入：Idempotency-Key、進行中請求、同業務唯一鍵（2026-11-27 warm-up）

**面試題**：下單 API 用 Header `Idempotency-Key`，實現係 Redis `SET key NX EX 86400`，set 成功就落單，失敗就當重複並試圖返回上次結果。Redis 往往只記咗「見過」，冇存完整 response。DB 對 `client_oid` 只係 `SELECT` 再 `INSERT`，冇唯一索引。用戶連點、請求超時後用同一 key 重試、或者換一個新 key 再用同一個 `client_oid` 提交，會發生咩事？點設計先正確？

- **冪等要有持久、可返回嘅記錄，唔只係一把鎖**。`SET NX` 只能擋住**同時**進來嘅第二下。紀錄要包含：請求指紋（body hash）、狀態（`PROCESSING`／`SUCCEEDED`／`FAILED`）、同成功時嘅 response。只存 `1`，重試時你返回唔到第一次嘅訂單號，前端會以為失敗再打。
- **Redis 唔可以做唯一真相**。過期、evict、主從切換／重啟之後 NX 再成功，就會再落一筆。TTL 太短（例如 30s）時，用戶超時重試好容易踩中。最終以 **DB 唯一約束** 為準，Redis 只係加速讀已完成結果。
- **同一 key、不同 body** 應回 422／衝突，唔好當同一筆。要比對請求指紋。
- **`SELECT` 再 `INSERT` 冇唯一索引**：兩個請求一齊查都係空，兩邊都插入 → 連點下兩次單。無論有冇 Idempotency-Key，`client_oid`（或等價業務鍵）都要有唯一索引，撞到就返回已有訂單。
- **進行中**：第一次還在處理（例如扣款 8 秒），客戶端 3 秒超時用同一 key 重試。正確係見到 `PROCESSING` 就等或回 409／「處理中」，**唔好再扣一次**，亦唔好喺第一次未寫入成功結果之前就用「重複」蓋掉。第一次失敗而且確認冇副作用，先允許同一 key 重試；已有副作用就要靠狀態機同對賬，唔能當無單。
- **Idempotency-Key ≠ 業務唯一鍵**。Key 解決「同一個 HTTP 嘗試被重放」（超時、連點、網關重試）。用戶超時之後換一個新 key、但 `client_oid` 相同，只靠 Idempotency-Key 擋唔到。相反，只有 `client_oid` 擋唔到「未生成 oid 就重放」嘅純 HTTP 重試。兩層一齊要：Key 去重請求，`client_oid`／`txHash` 去重業務。
- **鎖替代唔到已完成記錄**。鎖釋放後重試再進來，冇終態記錄仍會再執行。

### 追問

- 充值回調用 `txHash`（或唯一 deposit id），唔好用「金額 + 地址」。
- 出金回調嘅 `txHash` 同用戶「再提交」嘅 Idempotency-Key 各擋一類重複，唔能只用其中一個。
- 寫接口嘅冪等鍵要帶上 user／apiKey 範圍，避免跨用戶撞 UUID。
