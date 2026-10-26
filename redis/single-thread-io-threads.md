# Redis 單線程點解咁快？6.0 io-threads 改咗乜？

## 面試題
Redis 係單線程，點解仲咁快？6.0 之後仲算唔算單線程？

## 答案要點
- **「單線程」講嘅係命令執行**：讀寫數據由一條主線程順序做。持久化（BGSAVE/AOF rewrite 用 fork 子進程）、`UNLINK`/lazyfree、AOF fsync 由後台線程做。
- **快嘅原因**：
  - 純內存操作，冇磁碟隨機 IO。
  - IO 多路復用（epoll）：一條線程管幾萬條連接，邊條 socket 有數據就處理邊條。
  - 冇鎖、冇線程切換，命令天然原子。
  - 數據結構高效：SDS、listpack、skiplist、hashtable 漸進式 rehash。
- **真正瓶頸**通常係網絡 IO 同單核 CPU，唔係內存。
- **6.0 io-threads**：多線程只負責「讀 socket + 解析協議」同「寫回應」，**命令執行仍然單線程** → 原子性唔變，業務唔使加鎖。預設 `io-threads 1`（即關），讀都多線程要另開 `io-threads-do-reads yes`。只有高 QPS、網絡係瓶頸先值得開，線程數少過 CPU 核數（例如 4 核開 2–3）。

## 常見追問 / 陷阱
- **單線程 = 一條慢命令卡死全部人**：`KEYS *`、大 key `DEL`、大 Hash `HGETALL`、長 Lua、O(N) 命令。改用 `SCAN`/`HSCAN`、`UNLINK`、拆 key；用 `SLOWLOG GET` 搵兇手。io-threads 救唔到慢命令，因為執行仍然喺主線程。
- **「每條命令原子」≠「多條命令原子」**：`GET` 再 `SET` 中間可以被其他 client 插隊 → 用 `INCR`、`SET NX`、Lua 或 `MULTI/EXEC`。
- 想用多核：開多個實例 / Redis Cluster 分片，而唔係指望單實例多線程。

相關：[big key](big-key.md)、[熱 key](hot-key-rebuild-stampede.md)
