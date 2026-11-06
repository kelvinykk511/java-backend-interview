# Redis Hash 與 Pipeline

## 面試問題

什麼場景用 Redis `Hash` 比用一堆獨立 `String` key 更好？`Pipeline` 解決什麼問題？

## 答法

- **Hash**：一個 key 下多個 field→value（物件字段、用戶資料片段、配置項）。
  - 優點：一次 `HGETALL` / 批量 `HMGET`；比 N 個獨立 key 更好管理、常更省記憶體（小 field 壓縮）。
  - 典型：用戶 profile 片段、訂單狀態欄位、幣對行情的多個欄位。
- **別濫用**：超大 Hash（field 過多）會變成大 key；要掃描 / 過期單一 field 時，Hash 的 TTL 是整 key 級，field 不能單獨設過期（需業務自行處理或拆 key）。
- **Pipeline**：客戶端一次把多條命令打包送出，再一次讀回多個回應，減少 **RTT 往返**。
  - 適合：批量寫入、批量讀、初始化預熱。
  - **不是事務**：中間不會原子全成全敗；要原子用 `MULTI/EXEC` 或 Lua。
- **面試一句話**：Hash 管「一個實體的多個欄位」；Pipeline 砍網路來回，但不保證原子。

## 常見追問 / 陷阱

「Pipeline 和 `MGET` / Lua 怎麼選？」

→ 同類型批量讀優先原生批量命令；跨命令、要邏輯判斷用 Lua；只是「少幾次 RTT、命令彼此獨立」用 Pipeline。Pipeline 太大可能撐爆輸出緩衝或超時，要分批。

## 小練習

要把 100 個用戶的 `name`、`status` 從 Redis 撈出來，用 100 次 `HGET`、一次 Pipeline 包 100 次、還是別的做法？你會怎麼選並一句話說明理由？

---

## 深入（2026-11-06）：Hash 編碼、Pipeline 非原子、同連接佔用

### 面試問題

用戶錢包餘額（多幣種）／profile 多 field 點存？Pipeline 批量寫同 `MULTI/EXEC`、一次 `HMGET` 有咩分別？Pipeline 太大會點爆？

### 答法

**Hash 適用同限制**

- **適合金額、profile、配置**：一個 `user:{id}` Hash，field = `USDT` / `BTC` / `name` / `kycStatus`……一次 `HMGET` 攞多個 field，語意清晰。
- **小 Hash 省內存**：field／value 都短且數量少時，Redis 用 **listpack / ziplist** 緊湊編碼；超過 `hash-max-listpack-entries` / `hash-max-listpack-value`（或舊 ziplist 閾值）就升級成 hashtable，內存同 CPU 變差。
- **大 Hash = big key**：上萬 field 嘅 `HGETALL` 阻塞事件循環、主從複製／遷移痛苦；要拆 key（按業務分片）或改用 String + 前綴。見 [big-key](big-key.md)。
- **field 冇獨立 TTL**：`EXPIRE` 只作用喺整個 key。單一 field 要過期 → 自己喺 value 加過期時間戳判斷，或拆成獨立 String key。

**Pipeline vs 原生批量 vs 事務**

| 手段 | 減 RTT | 原子 | 說明 |
|---|---|---|---|
| 多次單命令 | ❌ | 每條自身原子 | 最慢 |
| **Pipeline** | ✅ | ❌ 非事務 | 打包送出；中間別人命令可插入；部分成功部分失敗都可能 |
| `HMGET` / `HGETALL` / `MGET` | ✅（一條命令） | 該命令原子 | **同 key 多 field 優先 HMGET**，唔好 Pipeline 包 N 次 `HGET` |
| `MULTI/EXEC` | 視實現 | ✅ 隊列原子執行 | 樂觀鎖常配 `WATCH`；見 [transaction](transaction-multi-vs-lua.md) |
| Lua | ✅ | ✅ 腳本內原子 | 要邏輯判斷、條件寫入用呢個 |

- Pipeline **唔係事務**：只係客戶端緩衝／少 RTT。要「扣餘額 + 寫流水」原子 → Lua 或 `MULTI/EXEC`，唔係 Pipeline。
- 對**同一個 Hash** 撈多個 field：一次 `HMGET key f1 f2 f3` 優於 Pipeline 打三條 `HGET`（命令數少、服務端更省）。
- Pipeline 適合：**跨好多唔同 key**、命令彼此獨立、只想減往返（例如預熱 1 萬個 key 嘅 `SET`）。

### 追問／陷阱

- **Pipeline 太大**：一次塞幾萬條 → 客戶端／伺服器 **output buffer** 爆、超時、連接被殺。要**分批**（例如每 500～2000 條 flush 一次），同監控 `client-output-buffer-limit`。
- **同步 Pipeline 佔住連接**：Jedis 傳統 sync pipeline 期間連接被佔，連接池變相變細；Lettuce 异步／reactive 較唔會堵死，但仍要限併發同批次大小。高併發下連接池大小同 pipeline 批次要一齊估。
- **Cluster**：Pipeline 入面 key 要落喺同一 slot（或讓客戶端按 slot 分組），否則 `MOVED`／跨槽失敗。
- 小練習參考答：100 用戶各要 `name`+`status` → **100 次 `HMGET user:{id} name status`**，再用 Pipeline **分批**包呢 100 條（或 Redis 7+ 有需要再考慮其他）；唔好 200 次單 `HGET`，亦唔好假設 Pipeline = 事務。

## 關聯

- [Big Key](big-key.md)
- [MULTI vs Lua](transaction-multi-vs-lua.md)
