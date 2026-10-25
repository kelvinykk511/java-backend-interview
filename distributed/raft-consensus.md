# Raft：選主同日誌複製

## 面試題
用簡單嘅話講 Raft 點樣選 leader、點樣保證已提交嘅數據唔會丟？（etcd、Kafka KRaft、Consul 都用佢）

## 答案要點
- **三種角色**：follower / candidate / leader。`term`（任期）單調遞增，好似「第幾屆」。
- **選舉**：
  - follower 喺 election timeout（隨機，例如 150–300ms，隨機係為咗避免大家同時參選）內收唔到 leader 心跳 → `term+1`，變 candidate 拉票。
  - 每個節點每個 term **只投一票**，而且只投畀日誌「至少同自己一樣新」嘅候選人（先比最後一條日誌嘅 term，再比 index）。
  - 攞到**過半票**就當選，即刻發心跳。
- **日誌複製**：
  - 寫請求只去 leader → 寫自己日誌 → 並行 `AppendEntries` 發畀 follower。
  - **過半節點落盤**就算 commit → apply 到狀態機 → 回覆 client；commit index 隨下一次心跳通知 follower。
  - `AppendEntries` 帶 `prevLogIndex/prevLogTerm`，對唔上就往前退再試；follower 衝突嘅日誌會被 leader 覆蓋（leader 永遠唔刪自己嘅日誌）。
- **點解已 commit 唔會丟**：commit 要過半有，當選又要過半票且日誌要夠新 → 兩個「過半」必有交集 → 新 leader 一定有齊所有已 commit 日誌。
  - 細節：leader 只會用「當前 term」嘅日誌計過半嚟 commit（論文 Figure 8），所以新 leader 上任先寫一條 no-op。
- **節點數**：3 個容 1 個壞、5 個容 2 個壞；偶數冇額外好處。跨機房常見 2+2+1。
- **對比你熟嘅**：Redis 主從 / Sentinel 係異步複製，failover 會丟寫；Kafka ISR 唔係多數派（minISR 控制）；ZooKeeper 嘅 ZAB 同 Raft 好似。

## 常見追問 / 陷阱
- **網絡分區時舊 leader 仲收唔收寫？** 會收，但永遠湊唔夠過半 → commit 唔到 → client 超時。新 leader 喺多數派嗰邊選出；網絡恢復後舊 leader 見到更高 term 自動降做 follower，未 commit 嘅日誌被覆蓋。
- **讀直接讀 leader 會唔會讀到舊值？** 會：佢可能已經被取代都唔知。解法：
  - **ReadIndex**：讀之前先同過半節點確認自己仲係 leader（etcd 預設 linearizable read）。
  - **Lease read**：靠租約時間，快啲但依賴時鐘。
  - etcd 嘅 serializable read 可以由任何節點讀，快但可能舊。
- **PreVote**：被隔離嘅節點不停自增 term，恢復後會打擾集群；PreVote 先試探「有冇機會贏」先真正加 term。
