# Redis 分布式鎖：SET NX PX、續期與 Redisson

**Q:** 用 Redis 實現分布式鎖，最少要處理邊幾個問題？

## 要點
- 加鎖要**原子**：`SET key uniqueValue NX PX 30000`（唔好 SETNX + EXPIRE 分兩步，中間掛咗就死鎖）。
- 解鎖要**驗證持有者**：value 用 UUID+線程ID，Lua 腳本「比較再 DEL」，避免刪咗別人的鎖。
- **鎖過期但業務未完成**：Redisson watchdog 每 10s 續期到 30s（只喺冇指定 leaseTime 時生效）。
- 可重入：Redisson 用 Hash 記錄 持有者→重入次數。
- 主從切換丟鎖：主節點寫入未同步就掛 → 兩個客戶端同時持鎖。RedLock 有爭議；真正強一致要求用 ZooKeeper/etcd，或者業務層加 **fencing token / 版本號 / DB 唯一約束** 兜底。

## 追問 / 陷阱
- 「鎖超時後業務仲寫咗 DB 點算？」→ 鎖只係效率保護，正確性靠冪等鍵或樂觀鎖（version）。
- 指定 leaseTime 會令 watchdog 失效，GC 長停頓可能令鎖提前過期。
