# Redis 分散式鎖

## 面試問題

用 Redis 做分散式鎖，最小可用方案是什麼？面試官常問的坑有哪些？

## 答法

- **最小可用**：`SET key uniqueValue NX EX seconds`（原子：不存在才設 + 過期）。解鎖用 Lua：比對 value 是自己的才 `DEL`，避免誤刪別人的鎖。
- **為什麼要唯一 value**：鎖過期後別人拿到鎖，你的延遲線程再 `DEL` 會把別人的鎖刪掉。
- **過期時間**：業務必須 **可重入/可續期** 時，才考慮看門狗（如 Redisson）；否則把 TTL 設得比最壞執行時間長一點，並保證冪等。
- **正確性邊界**：單 Redis 節點簡單；主從切換可能丟鎖（主寫了鎖還沒同步就掛）。強一致場景要談 **RedLock 爭議** 或改用 ZooKeeper / etcd / DB 唯一約束。
- **CEX 角度**：下單、劃轉等關鍵路徑，鎖只是互斥輔助，**最終仍靠 DB 唯一鍵 / 狀態機冪等**，不能只靠 Redis 鎖。

## 常見追問／陷阱

「鎖住了但業務還沒做完，TTL 到期怎麼辦？」

→ 續期（watchdog）或把臨界區拆短 + 業務冪等；千萬不要「TTL 設永久」——進程掛了就死鎖。

## 關聯

- [Redisson 續期與可重入細節](../redis/distributed-lock-redisson.md)
- [介面與消息的冪等](idempotency.md)

---

## 深入（2026-11-21）：Redisson watchdog、主從丟鎖、fencing token

### 面試可以點追

「`SET NX PX` 夠唔夠？Redisson 自動續期點運作？指定 `leaseTime` 同唔指定有咩分別？主從切換兩個客戶端同時持鎖點辦？出金／對帳可唔可以只靠 Redis 鎖？」

### 答法

**最小正確用法（仍然係面試底線）**

1. 加鎖：`SET lock:withdraw:{uid} {token} NX PX {ttl}`（token = UUID + 線程／進程標識）。
2. 解鎖：Lua `if get==token then del`，**唔好**只 `DEL`。
3. 唔好 `SETNX` 再 `EXPIRE`——兩步之間掛咗就永久鎖死。

**Redisson 常見行為（講得出就加分）**

- **未指定 leaseTime**：拿到鎖後，watchdog 大約每 `lockWatchdogTimeout/3`（預設 30s → 約 10s）續期；進程活著就一直續，掛咗先靠 TTL 自動釋放。
- **指定 leaseTime**：到期**唔會**自動續；GC 長停頓／慢 DB 令臨界區超過 lease → 別人搶到鎖，你仲以為自己持有，繼續寫 → **雙執行**。
- **可重入**：同一線程用 Hash 記重入次數；解鎖要配對次數，否則提前釋放。
- **可重入 ≠ 跨進程安全**：另一個 JVM 唔會共享你嘅重入場次。

**主從／Cluster 丟鎖**

- 鎖寫喺 master，未複製到 replica，master 掛 → replica 升主，鎖「消失」→ 第二個客戶端 `SET NX` 成功 → **兩人同時持鎖**。
- RedLock（多獨立節點多數派）有學術爭議（時鐘、GC、網絡分區）；面試態度：**講得出風險**，資金路徑唔好只押 RedLock。
- 更穩：ZooKeeper／etcd（session + fencing）、或者 **根本唔把正確性放喺鎖上**。

**Fencing token／業務兜底（CEX 必講）**

- 每次加鎖拿到遞增 token（或用 DB `UPDATE ... WHERE version=?`／唯一鍵 `INSERT`）。
- 寫資源時帶上 token／version；存儲拒絕過期 token → 就算「兩把鎖」都只有一個寫得入。
- 出金、入帳、對帳：**Redis 鎖 = 減併發／防熱點**；**正確性 = DB 唯一約束 + 狀態機**。鎖過期後舊線程仲寫 DB，若冇 version／狀態條件，照樣錯帳。

### 追問／陷阱

- 「TTL 設永不過期最安全？」→ 進程／線程死咗就死鎖；運維要人手刪。一定要 TTL +（可選）續期。
- 「鎖住就一定互斥成功」→ 否。主從切換、lease 過短、誤刪別人鎖、臨界區做 RPC 拖長，都會破功。
- 「用 `Thread.sleep` 到業務估時再續期」→ 自研易漏；寧用成熟庫或短臨界區 + 冪等。
- 熱 key：`lock:hotcoin` 全集群搶一把 → Redis CPU 同超時；改成更細粒度（uid／訂單號）或本地隊列串行。
