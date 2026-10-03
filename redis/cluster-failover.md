# Redis Cluster Failover

## 面試問題

Redis Cluster 某個 master 掛了，failover 怎麼走？客戶端會看到什麼？跟單機 Redis 差在哪？

## 答法

**角色與偵測**

- 每個 hash slot 由一個 **master** 負責寫；通常配 **replica** 異步複寫。
- 節點彼此心跳；超過 **`cluster-node-timeout`** 多數派認為該 master 失敗（PFAIL → FAIL）→ 進入 failover。

**選主（簡化）**

1. 該 master 的 replica 們發起選舉（帶 epoch）。
2. 多數 master 投票；勝出的 replica **升主**，接管原 slot。
3. 叢集配置 epoch 遞增並廣播新拓撲。

**客戶端影響**

- 打到舊節點／錯節點：常見 **`MOVED`**（永久重定向，slot 已歸新主）或連線失敗後重連。
- **`ASK`**：遷移中的臨時重定向（reshard），跟 failover 的 `MOVED` 不同，別混。
- SDK 必須能**刷新 slot map**、重連；否則會持續打錯節點或超時。
- Failover 窗口內該 slot **可能短暫不可寫**。

**寫入安全**

- 預設**異步**複寫：剛寫進舊 master、尚未傳到 replica 的資料**可能丟失**（跟異步主從同一取捨）。

**vs 單機 Redis（面試常踩）**

| | 單機 | Cluster |
|---|---|---|
| 故障 | 整實例掛＝全掛（或靠哨兵另套） | 單 master 掛 → 該 slot 的 replica 升主 |
| 客戶端 | 連一個 endpoint | 要懂 slot／MOVED／拓撲刷新 |
| 多 key | 任意 | **同 slot**（或 hash tag）才能跨 key 原子／事務 |
| 一致性 | 單點無複寫丟窗 | 異步複寫仍有極短丟窗 |

**面試一句話**：超時被多數派判定 FAIL → replica 選舉升主並廣播 slot；客戶端靠 MOVED／刷新拓撲接上，並接受異步複寫下的極短丟資料風險。

## 常見追問 / 陷阱

「開了 replica 就一定不丟資料？」

→ 否。預設異步；要更強需業務層（寫後確認、或接受更高延遲）。另一陷阱：維護／手動 `CLUSTER FAILOVER` 後客戶端快取舊拓撲 → 錯路由或超時。還有：把 Cluster 當單機用（跨 slot 事務／`KEYS`／單連接假設）會踩坑。

## 小練習

CEX 熱路徑用 Redis Cluster 做成交緩存。自動 failover 後部分鍵讀到舊值、部分超時。你會先查節點／epoch、客戶端 slot 快取，還是業務 TTL？為什麼？
