# Redis 事務 MULTI/EXEC vs Lua 腳本

## 面試問題

Redis 嘅 `MULTI/EXEC` 算唔算事務？同 Lua 腳本比，「扣庫存」或者「限流」應該用邊個？

## 答法

- **MULTI/EXEC 做到咩**：命令先排隊，`EXEC` 時一次過連續執行，中間唔會插入其他客戶端命令（隔離）。
- **做唔到咩**：
  - **冇回滾**：排隊時語法錯（命令名錯）→ 成個事務放棄；但執行時錯（例如對 String 做 `LPUSH`）→ 嗰條失敗，**其他照樣執行**。
  - **唔可以「先讀再判斷」**：排隊期間攞唔到結果，冇得寫 `if 庫存 > 0 then 扣`。
  - 要條件判斷就加 `WATCH`（樂觀鎖）：被 watch 嘅 key 喺 EXEC 前被改 → EXEC 返回 nil，自己重試。高並發下重試多、浪費。
- **Lua 腳本**：成個腳本喺 Redis 單線程入面一次過跑完，**可以讀、判斷、再寫**，期間冇其他命令插入。扣庫存、限流、釋放分布式鎖（比較 value 先 DEL）都用 Lua。
  - 但 Lua **都冇回滾**：跑到一半 `redis.call` 報錯，之前嘅寫入唔會撤銷。要自己先檢查、後寫。
- **實戰寫法**：
  - 用 `EVALSHA` + 緩存 SHA；遇到 `NOSCRIPT`（重啟/切主後腳本冇咗）就 fallback `EVAL`。Spring 嘅 `DefaultRedisScript` 已經幫你做。
  - 所有 key 經 `KEYS[]` 傳入，唔好喺腳本入面拼 key。
  - Redis 7 可以用 **Functions**（`FUNCTION LOAD`），會持久化同複製，唔怕 NOSCRIPT。

## 常見追問 / 陷阱

- **腳本跑太耐**：Lua 執行期間 Redis 處理唔到任何請求，全部客戶端卡住。超過 `lua-time-limit` / `busy-reply-threshold`（預設 5 秒）其他客戶端收到 `BUSY`，只能 `SCRIPT KILL`（腳本未寫過嘢先殺到）；已經寫過就只得 `SHUTDOWN NOSAVE`。→ 腳本要短，唔好喺入面循環大 key。
- **Cluster 跨 slot**：腳本或 MULTI 涉及嘅 key 必須喺同一個 slot，否則 `CROSSSLOT`。用 hash tag：`stock:{sku1}`、`order:{sku1}`。
- **Pipeline ≠ 事務**：pipeline 只係減少網絡往返，命令之間可以被其他客戶端插隊，亦唔原子。
- 腳本要確定性：Redis 7 預設按效果複製（effects replication），但仍唔好依賴隨機值做業務判斷。

## 一句話

MULTI = 打包連續執行、冇回滾、冇得判斷；Lua = 可以判斷嘅原子操作、都冇回滾、要短、Cluster 要同 slot。
