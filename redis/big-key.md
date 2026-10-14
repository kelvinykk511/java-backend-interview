# Redis Big Key：點發現、點拆、點刪

## 面試問題

線上 Redis 偶爾 RT 飆高、甚至主從切換，懷疑係 big key。你點樣定位？搵到之後點處理？

## 答法

**咩叫 big key（經驗值）**

- String value > ~10KB（嚴格啲 > 1MB 一定有問題）。
- Hash/List/Set/ZSet 元素 > 5000～1 萬，或者總大小好大。

**點解危險**

- Redis 單線程執行命令：`HGETALL`、`DEL`、`LRANGE 0 -1` 一次處理大量數據 → **阻塞其他請求**。
- 網卡打滿（一個 key 幾 MB × 高 QPS）。
- Cluster 數據/流量**傾斜**，某個 slot 嘅節點特別熱。
- 過期/淘汰時刪除大 key 都會阻塞；主從同步、遷移 slot 都慢。

**點發現**

- `redis-cli --bigkeys`（SCAN 抽樣，每種類型揾最大；只計元素數）/ `--memkeys`（Redis 7，按內存）。
- `MEMORY USAGE key`、`OBJECT ENCODING`。
- 離線分析 RDB（如 `rdb-tools`、`redis-rdb-cli`），唔影響線上。
- 監控：slowlog、網卡流量、單節點內存傾斜。
- **唔好用 `KEYS *`**。

**點處理**

- **拆**：大 Hash 按 `hash(field) % N` 拆成 `user:{id}:0..N-1`；大 List/ZSet 按時間或分片拆。
- **讀**：用 `HSCAN`/`SSCAN`/`ZSCAN` 分批，或者只拎需要 field（`HMGET`），唔好 `HGETALL`。
- **刪**：用 `UNLINK`（後台線程釋放內存）；舊版本用 SCAN 分批 `HDEL`/`LTRIM` 慢慢刪。
- 開 `lazyfree-lazy-expire`、`lazyfree-lazy-eviction`、`lazyfree-lazy-server-del`，避免過期/淘汰時阻塞。
- **源頭**：設計時限制集合大小，壓縮 value（或唔好放 Redis），寫入前校驗大小。

## 常見追問／陷阱

- 「`DEL` 一個 1 萬元素嘅 Hash 有幾慢？」`DEL` 對集合係 O(N)，元素多就阻塞主線程；`UNLINK` 只係 O(1) 解除引用，真正釋放交俾後台。
- 「big key 同 hot key 分別？」big key 係**大**（單次操作貴），hot key 係**熱**（QPS 集中）。可以同時出現，處理方法唔同：hot key 靠本地緩存/多副本 key。
- 陷阱：`--bigkeys` 只報每種類型最大嗰個、按元素數計，唔等於「全部 big key 清單」。
- 交易所例子：成個交易對嘅 orderbook 或者某用戶全部訂單塞入一個 Hash → 行情高峰一個 `HGETALL` 就拖慢整個節點。

## 相關

- [熱 key / 緩存重建擊穿](hot-key-rebuild-stampede.md)
- [過期與淘汰](expiration-eviction.md)
