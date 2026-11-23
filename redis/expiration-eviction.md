# Redis 過期刪除 vs 內存淘汰

> 面試題：一個 key 設了 `EX 60`，60 秒後它一定馬上從內存消失嗎？內存滿了 Redis 又會怎樣？

## 兩件事，別混淆

| | 過期刪除（expire） | 內存淘汰（eviction） |
|---|---|---|
| 觸發 | key 到期 | 用量超過 `maxmemory` |
| 刪誰 | 已過期的 key | 按策略挑「還沒過期」的 key 也會刪 |
| 目的 | 實現 TTL 語義 | 保護內存不爆 |

## 過期刪除：惰性 + 定期

- **惰性刪除**：讀/寫某 key 時先檢查是否過期，過期就刪並當作不存在 → 到期後一定讀不到。
- **定期刪除**：後台每秒多次（`hz` 默認 10）隨機抽樣帶 TTL 的 key，刪掉過期的；若過期比例高就繼續抽，但有時間上限，避免卡主線程。
- 所以「讀不到」是準時的，但「內存釋放」不是：沒人訪問的過期 key 可能多留一陣。
- 主從：從庫不主動刪過期 key，由主庫刪後同步 `DEL`；從庫讀時會判斷過期返回空（3.2+）。

## 內存淘汰策略（`maxmemory-policy`）

- `noeviction`（默認）：寫入報錯 `OOM command not allowed`，讀照常。
- `allkeys-lru` / `allkeys-lfu`：所有 key 中挑最近最少用 / 最不常用的刪。**純緩存首選**。
- `volatile-lru` / `volatile-lfu` / `volatile-ttl` / `volatile-random`：只在「設了 TTL」的 key 裡挑。
- `allkeys-random`。
- LRU/LFU 都是**近似**：抽樣 `maxmemory-samples`（默認 5）個挑最差的，不是精確鏈表。
- LFU（4.0+）：計數會隨時間衰減，比 LRU 更能抵抗「一次性批量掃描把熱 key 擠走」。

## 實戰判斷

- 純緩存：`allkeys-lfu` 或 `allkeys-lru`，配合所有 key 都有 TTL。
- 緩存 + 不能丟的數據（鎖、計數、隊列）混放：用 `volatile-*` 只淘汰有 TTL 的；**更好是分實例**。
- 坑：選了 `volatile-*` 但大多數 key 沒 TTL → 沒得淘汰，行為等同 `noeviction`，寫入開始報 OOM。
- 坑：大量 key 同一時刻過期 → 定期刪除忙 + 緩存雪崩；TTL 加隨機抖動。
- 坑：`maxmemory` 不設（64 位默認無上限）→ 被系統 OOM killer 殺；還要預留 fork（RDB/AOF rewrite）的 COW 內存。

## 常見追問

- **淘汰會不會刪掉分布式鎖的 key？** `allkeys-*` 下會 → 鎖提前消失，兩個人同時拿到鎖。所以鎖不要和大緩存放同一個 allkeys 實例，且業務層仍要冪等。
- **怎麼監控？** `INFO stats` 的 `evicted_keys`、`expired_keys`；`used_memory` vs `maxmemory`；`evicted_keys` 持續增長 = 容量不夠或有大 key。

## 深入：miss 係過期定淘汰（2026-11-23 warm-up）

**面試題**：訂單緩存突然大面積 miss，有人話 TTL 到咗；但 `evicted_keys` 急升、`expired_keys` 正常，出金鎖又偶發提早消失。你點判斷？policy 點配？

- **先分兩條計數**。`INFO stats` 嘅 **`expired_keys`** 係 TTL 到期被刪；**`evicted_keys`** 係用量超過 `maxmemory`，按 `maxmemory-policy` 趕走——**未過期都會刪**。呢題係淘汰，唔係到期。
- **排查順序**：`used_memory` 對 `maxmemory`、`evicted_keys`／`expired_keys` 增速、有冇大 key、fork／COW 有冇再抬高內存。miss 唔等於「EX 到咗」。
- **`allkeys-lru`／`allkeys-lfu`** 會淘汰**所有** key，包括出金鎖、限流計數。鎖提早沒 → 另一台攞到鎖，重複出金。鎖同大緩存**分實例**；業務層仍然要有唯一鍵／狀態機，唔好當 Redis 鎖係正確性保證。
- **改 `volatile-lru` 唔等於鎖安全**：`volatile-*` 只喺**設咗 TTL** 嘅 key 裡挑，`SET NX PX` 嘅鎖正在呢個集合，一樣會被淘汰。它保住嘅係**冇 TTL** 嘅資料（例如常駐配置）。
- **`volatile-*` + 大量無 TTL key**：可淘汰嘅集合太細，釋放唔到 `maxmemory` 以下，寫入報 `OOM command not allowed`，行為接近 `noeviction`。
- **過期但內存仍高**：讀唔到係準時嘅（惰性刪），內存釋放靠定期抽樣，冷嘅過期 key 會滯留。可對照 `expired_keys` 同內存用量；必要時 `lazyfree-lazy-expire`／`lazyfree-lazy-eviction`（非同步釋放大對象，唔等於唔使淘汰）。
- **純緩存**：`allkeys-lfu`（或 lru）+ 每個 key 都有 TTL（加抖動）；`maxmemory` 一定要設，並預留 fork 嘅 COW。持久資料同鎖分開實例。

### 追問

- 從庫預設跟主庫刪除（`replica-ignore-maxmemory` 預設 yes），唔好假設從庫會獨立按 `maxmemory` 淘汰。
- LRU／LFU 係抽樣近似，唔係精確鏈表；一次大掃描可以擠走熱 key，LFU 會好啲但仍要監控 `evicted_keys`。
