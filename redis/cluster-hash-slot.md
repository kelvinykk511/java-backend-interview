# Redis Cluster 與 Hash Slot

## 面試問題

Redis Cluster 怎麼把 key 分到各節點？為什麼會出現「跨 slot 不能一次多 key」的限制？

## 答法

- **Hash Slot**：整個叢集固定 **16384** 個 slot；`CRC16(key) % 16384` 得到 slot，再查「哪個 master 負責這個 slot」。
- **擴容／縮容**：搬的是 slot（連帶上面的 key），不是「整庫複製」；客戶端靠 `MOVED` / `ASK` 重定向到正確節點。
- **多 key 命令**：`MGET`、`DEL` 多 key、事務、Lua 裡多 key，**所有 key 必須在同一 slot**，否則報 `CROSSSLOT`。
- **Hash Tag**：用 `{...}` 指定只對括號內字串算 slot，例如 `user:{1001}:profile` 與 `user:{1001}:orders` 同 slot，方便同用戶的相關 key 一起操作。
- **面試一句話**：Cluster 用 16384 slot 路由；跨 slot 的多 key 操作要靠 Hash Tag 或拆成單 key／業務層組裝。

## 常見追問 / 陷阱

「為什麼是 16384 不是更多？」

→ 節點心跳要帶 slot 配置位圖；16384 在「可擴展」與「心跳訊息大小」之間折衷。實務更常被問：大 key、熱點 slot、以及沒用 Hash Tag 導致相關資料散落各節點。

## 小練習

訂單 key 是 `order:{oid}`、庫存是 `stock:{sku}`。若要在 Lua 裡「扣庫存 + 寫訂單」一次原子完成，key 設計要怎麼改才不會踩 `CROSSSLOT`？

---

## 進階（2026-11-04）：`MOVED` vs `ASK`，slot 遷移期間點路由

### 面試問題

Redis Cluster 擴容，slot 5000 正由節點 A 搬去 B。遷移期間客戶端讀寫呢個 slot 嘅 key 會發生咩？`MOVED` 同 `ASK` 有咩分別？

### 答法

- **`MOVED`**：slot **已經永久**屬於另一個節點。客戶端要**更新本地 slot 表**，之後直接去新節點。
- **`ASK`**：slot **遷移中**（A 標記 `MIGRATING`、B 標記 `IMPORTING`）。客戶端去 A：
  - key 仲喺 A → A 直接處理；
  - key 已經搬走 → A 回 `ASK 5000 B`，客戶端去 B **先發 `ASKING`** 再發命令，**只係呢一次**，**唔更新** slot 表（slot 未正式交接）。
- **點解要 `ASKING`**：B 未正式擁有 slot，冇 `ASKING` 會回 `MOVED` 指返 A，形成來回彈。
- **遷移完成**：`CLUSTER SETSLOT 5000 NODE B` → 之後大家收到 `MOVED`，刷新 slot 表。
- 智能客戶端（Lettuce / Jedis Cluster / Redisson）會自動處理兩種重定向，但要設**重定向次數上限**同 **topology refresh**（Lettuce 預設唔開週期刷新，failover 後可能長時間打錯節點）。

**面試一句話**：`MOVED` 係「搬咗家，記住新地址」；`ASK` 係「搬緊，今次先去嗰邊問（加 ASKING），唔好改地址簿」。

### 追問 / 陷阱

- **多 key / Lua 喺遷移中**：如果部分 key 搬咗、部分未搬，會回 `TRYAGAIN`，業務要重試——同一 hash tag 嘅 key 同 slot，但遷移期間仍可能「半搬」。
- **大 key 遷移**：`MIGRATE` 係同步搬，一個大 key 會阻塞源同目標節點，同 [Big Key](big-key.md) 一齊講。
- **Hash Tag 濫用**：所有 key 都用同一個 `{tag}` → 全部落同一 slot，cluster 等於冇分片，熱點節點打爆。
