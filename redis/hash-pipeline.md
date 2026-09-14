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
