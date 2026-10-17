# MySQL 死鎖排查實戰

> 基礎概念見 [InnoDB 死鎖：成因、檢測與避開](deadlock.md)

## 面試題

線上報 `ERROR 1213 Deadlock found when trying to get lock`，你點一步步排查同修？

## 標準答案

### 1. 拎證據

- `SHOW ENGINE INNODB STATUS\G` → `LATEST DETECTED DEADLOCK`（只保留**最近一次**）。
- 長期：`innodb_print_all_deadlocks=ON`，死鎖寫入 error log。
- MySQL 8.0：`performance_schema.data_locks` / `data_lock_waits` 睇**當前**鎖等待。

### 2. 讀 log

- 兩個事務各自 **HOLDS THE LOCK(S)** 同 **WAITING FOR THIS LOCK**。
- 睇 index 名、`lock_mode` X/S、`locks rec but not gap`（記錄鎖）、`locks gap before rec`（間隙鎖）、`insert intention`。
- InnoDB 回滾 **undo 量細**嗰個（`WE ROLL BACK TRANSACTION (n)`）。

### 3. 還原場景

搵返兩條 SQL 對應業務代碼，畫加鎖時序。常見四類：

| 類型 | 描述 |
|------|------|
| (a) | 兩事務**相反順序**更新同兩行（轉帳 A→B 同 B→A） |
| (b) | RR 下 `SELECT ... FOR UPDATE` 一個**唔存在**嘅值 → 兩邊都拎 gap lock（gap 之間唔互斥）→ 再 INSERT 要 insert intention 被對方 gap lock 擋 → 互卡 |
| (c) | 二級索引同主鍵**加鎖順序唔同**（一條走二級索引再回表、一條直接鎖主鍵） |
| (d) | update 條件**冇走索引**，掃描鎖大量行 |

### 4. 修

- **固定加鎖順序**（例如按 `account_id` 由細到大）。
- **縮短事務**：唔好喺事務入面做 RPC/HTTP。
- 確保 `WHERE` **走索引**。
- 「唔存在就插入」→ 唯一鍵 + `INSERT ... ON DUPLICATE KEY UPDATE`，或捕獲 duplicate key。
- 業務容許可考慮 **RC**（減少 gap lock，需 `binlog_format=ROW`）。
- 應用層對 **1213** 做**有限次**、**成個事務**重做嘅重試。

### 5. 死鎖 vs 鎖等待超時

- **1213 死鎖**：`innodb_deadlock_detect` 即時檢測並回滾一方。
- **1205 鎖等待超時**：等夠 `innodb_lock_wait_timeout`（預設 50s）。

## 追問・陷阱

- **關掉 `innodb_deadlock_detect` 會唔會好啲？** 熱點行高併發時死鎖檢測 CPU 成本會好高，有人會關，但要調細 `lock_wait_timeout` 兜底；更好係解決熱點本身（拆子帳戶、排隊串行化）。
- **重試只重試報錯嗰條 SQL 係錯**：1213 已經 rollback **成個事務**，要由事務開頭重做。

## 實戰 (CEX) 角度

- 資金帳戶轉帳／劃轉按 `account_id` **排序加鎖**。
- 熱門帳戶（例如手續費收入帳戶）用**分桶子帳戶**或**異步彙總**，避免熱點行。

## See also

- [InnoDB 死鎖：成因、檢測與避開](deadlock.md)
- [Gap / Next-Key 鎖](gap-next-key-lock.md)
- [RC vs RR](rc-vs-rr.md)
