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
