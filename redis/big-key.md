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

---

## 深入（2026-11-19）：orderbook／持倉大 Hash，HGETALL + DEL 重建

### 面試可以點追

行情把成個交易對 orderbook 塞進一個 Redis Hash（幾萬 field），開盤 `HGETALL` 期間 RT 飆高，甚至主從延遲、故障切換。對帳又會 `HGETALL` 再 `DEL` 重建用戶持倉 Hash。Big key 同 hot key 有咩分別？點定位、點拆、點刪？

### 答法

**先分清**

- **Big key**：單個 key **大**（value 大或元素極多），一次 `HGETALL`／`DEL` 就貴，阻塞單線程、打滿網卡、拖慢複製同 slot 遷移。
- **Hot key**：QPS **集中**喺同一個 key。熱門交易對嘅 orderbook 可以**又大又熱**，但治理方法唔同：大要拆同分批；熱要本地緩存／多副本 key／限流。只加本地緩存，救唔到 `DEL` 一個幾萬 field 嘅 Hash。

**點定位（唔好 `KEYS *`）**

- 線上：slowlog、單節點 CPU／網卡、Cluster 某節點內存或流量傾斜（一個大 key 只能喺**一個 slot／一個節點**）。
- `MEMORY USAGE key`、`OBJECT ENCODING`；`redis-cli --bigkeys`（SCAN 抽樣，**每種類型只報元素數最大嗰個**，唔係完整清單，亦唔係按內存）。Redis 7 可用 `--memkeys`。
- 完整盤點離線分析 RDB（`rdb-tools` 等），避免喺主庫掃描。

**點拆、點讀**

- Hash 按 `hash(field) % N` 拆成 `orderbook:{symbol}:{shard}` 或 `pos:{uid}:{shard}`；List／ZSet 按時間窗或分片拆。Cluster 要令拆出嚟嘅 key 分散到唔同 slot，單個大 key **拆唔開**同一個 slot。
- 讀用 `HSCAN`／`HMGET` 只拎需要嘅 field，禁止開盤路徑 `HGETALL` 成本帳。
- 設計期就要限制集合大小；實在大嘅快照唔好放 Redis。

**點刪（呢度最易踩坑）**

- 集合型 `DEL` 係 O(N)，幾萬 field 會卡住主線程，複製緩衝堆積、replica 追唔上，嚴重可以引發切換。
- 用 `UNLINK`（主線程 O(1) 解除引用，內存由後台釋放）；舊版本用 `HSCAN` + 分批 `HDEL`／`LTRIM`。
- 開 `lazyfree-lazy-expire`、`lazyfree-lazy-eviction`、`lazyfree-lazy-server-del`，避免過期／淘汰／rename 覆蓋時隱式大刪除。
- 對帳「整本重建」唔好 `HGETALL` + `DEL`：改分片寫入新 key，再原子切換（rename／指針 key），舊 key 用 `UNLINK`。

### 追問／陷阱

- `--bigkeys` 唔等於「全部大 key 清單」，元素少但 value 好大嘅 String 可能漏報；要配 `--memkeys` 或 RDB 分析。
- 大 key 刪除引發嘅「主從延遲／切換」，根因往往係主線程阻塞同複製落後，唔係複製本身突然壞。
- Pipeline 只係減 RTT，**唔會**把一條 `HGETALL`／`DEL` 變成非阻塞。
