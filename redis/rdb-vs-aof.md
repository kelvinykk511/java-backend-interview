# Redis 持久化：RDB vs AOF

## 面試問題

RDB 和 AOF 分別是什麼？怎麼選？`BGSAVE`／AOF rewrite 時為什麼會突然多吃一倍內存？

## 答法

- **RDB**：某個時間點的內存快照（`.rdb`）。恢復快、文件小；崩潰可能丟最後一次快照之後的數據。
- **AOF**：把寫命令追加到日誌。可配置 `always` / `everysec` / `no`；通常更「不丟數據」，文件更大，要靠 rewrite 壓縮。
- 實務常見：**RDB + AOF 都開**。重啟時若 AOF 存在，優先用 AOF 恢復（更新）。
- **fork + COW**：`BGSAVE`／AOF rewrite 會 `fork` 子進程。期間若寫很多，父進程頁被改 → Copy-on-Write 複製髒頁，RSS 可能逼近「再多一份」；大 key／寫高峰時要盯記憶體水位。
- 面試可補一句：主從／哨兵／集群是高可用；持久化解決的是「進程掛了數據還在不在」。

## 常見追問 / 陷阱

「AOF `everysec` 最多丟多少？」

→ 大約最多約 **1 秒** 的寫入（異步刷盤節奏），不是絕對零丟失。

## 小練習

CEX 熱路徑餘額若幾乎不能丟，你會偏 RDB、AOF，還是兩者 + 別的手段？為什麼單靠 Redis 持久化不夠？

## 深入：混合持久化同生產配置（2026-10-23 warm-up）

**面試題**：RDB 同 AOF 點揀？生產點配？

### RDB（快照）

- 某時刻快照；`BGSAVE` **fork** 子進程寫 `dump.rdb`，靠 **Copy-On-Write**。
- **優點**：文件細、恢復快、適合備份／全量同步。
- **缺點**：兩次快照之間嘅寫會丟（**分鐘級**）；fork 大內存實例會卡主線程（頁表複製，約 **10～20ms/GB** 級，視乎機器）；寫多時 COW 令內存最多**翻倍**。

### AOF（命令日誌）

- 每條寫命令追加；`appendfsync`：
  - **always**：最安全最慢；
  - **everysec**（預設）：後台線程每秒 fsync，通常最多丟約 **1～2 秒**；
  - **no**：交 OS，約 **30 秒** 級風險。

### `everysec` 陷阱（磁碟慢時）

- 若上次 fsync **超過 2 秒**未完成，主線程 `write` 會**阻塞等** → 最壞約丟 **2 秒**，磁碟慢時會拖慢寫延遲。
- 確認：`INFO persistence` 嘅 **`aof_delayed_fsync`** 計數；日誌 **`Asynchronous AOF fsync is taking too long`**。

### AOF rewrite

- `BGREWRITEAOF` fork 子進程按當前數據生成最小命令集；重寫期間新寫入入 **rewrite buffer**。
- **Redis 7**：改為 **multi-part AOF**（base + incr 文件 + manifest），唔再雙寫 buffer。
- 自動觸發：`auto-aof-rewrite-percentage`／`min-size`。

### 混合持久化（4.0+）

- `aof-use-rdb-preamble yes`（**7.0 預設開**）：重寫時 base 用 **RDB 格式** + 後面增量 AOF → 恢復快又丟得少。

### 啟動恢復

- 有開 AOF → **優先載 AOF**；否則 RDB。

### 生產建議

- **純緩存**：可以關持久化或只開 RDB。
- **要數據安全**：AOF `everysec` + **混合** + 定期 RDB 備份到異地。
- **主從**：可喺 **slave** 做持久化減主庫 fork 壓力（注意：主庫冇持久化 + 自動重啟會清空數據並同步到從庫嘅坑）。
- 預留內存 **headroom** 俾 COW；關 **THP**（透明大頁會放大 COW）。

### 追問

- **fork 點解會卡？** → fork 本身同步複製**頁表**，喺主線程；`INFO persistence` 嘅 **`latest_fork_usec`** 睇耗時。
- **AOF 文件損壞？** → `redis-check-aof --fix`；`aof-load-truncated yes` 允許截斷尾部載入。

### 注意

- 持久化 **≠ 唔丟數據**；主從異步複製 + failover 仍可能丟 → 見 [主從複製](master-replica-replication.md)（如存在）。
