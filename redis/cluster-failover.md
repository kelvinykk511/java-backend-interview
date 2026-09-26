# Redis Cluster Failover

## 面試問題

Redis Cluster 某個 master 掛了，流量會怎樣接上？客戶端會看到什麼？

## 答法

- **角色**：每個 hash slot 有一個 **master** 負責寫；通常還有 **replica** 做複寫。
- **故障偵測**：節點彼此心跳；多數派認為某 master 失敗 → 進入 failover。
- **選主**：該 master 的某個 replica 被選成新 master，接管原 slot；叢集配置（epoch）遞增並廣播。
- **客戶端**：寫到舊節點會收到 `MOVED`（或連線失敗後重連拓撲）；SDK 要能刷新 slot map，否則會打到錯節點。
- **寫入安全**：failover 期間可能短暫不可寫；異步複寫下，剛寫進舊 master、尚未傳到 replica 的資料**可能丟失**（類似異步複製的取捨）。
- **面試一句話**：Cluster 靠多數派選出 replica 升主並廣播新 slot 映射；客戶端必須處理重定向與拓撲刷新，且要接受異步複寫下的極短窗口丟資料風險。

## 常見追問／陷阱

「開了 replica 就一定不丟資料？」

→ 否。預設是**異步**複寫；要更強一致需業務層（例如寫後讀確認、或接受更高延遲的同步策略）。另一陷阱：人為 `CLUSTER FAILOVER`／維護時若客戶端快取舊拓撲，會出現一段時間的錯路由或超時。

## 小練習

CEX 熱路徑用 Redis Cluster 做成交緩存。一次自動 failover 後，部分鍵讀到舊值、部分超時。你會先查節點狀態、客戶端 slot 快取，還是業務 TTL？為什麼？
