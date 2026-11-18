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

---

## 深入（2026-11-18）：二級索引同主鍵加鎖順序（類型 c）

### 面試可以點追

訂單表一個事務用 `SELECT ... FROM orders WHERE user_id=? FOR UPDATE`（二級索引）再改狀態；另一個按主鍵 `UPDATE orders SET ... WHERE id=?`，有時仲會改到帶索引嘅列。同一行或相鄰幾行都會偶發 1213。點解？點從 `INNODB STATUS` 證實？改 RC 係咪就冇？

### 答法

**點解會成環**

- InnoDB 行鎖掛喺**索引記錄**上面。同一行經唔同索引進去，加鎖順序可以相反。
- 事務 A：`WHERE user_id=?` 當前讀，通常**先鎖二級索引記錄**，再回表鎖**聚簇索引（主鍵行）**。
- 事務 B：`WHERE id=?` 嘅 `UPDATE` **先鎖主鍵行**；若修改咗二級索引列（或要維護該二級索引），再去鎖／刪舊、插新二級記錄。
- 時序一對上：A 持有 `idx_user_id`、等 PRIMARY；B 持有 PRIMARY、等 `idx_user_id` → 環等待 → 1213。兩個事務處理同一個 user 下面多行、一個走二級一個走主鍵但擴鎖順序相反，同樣會中。
- 注意：B 如果只改非索引列，可能只鎖主鍵、唔再去搶二級鎖，噉同一行多數係 **1205 鎖等待**而唔係死鎖。死鎖通常要「兩邊都要對方手上嗰把索引鎖」。

**點證實**

- `SHOW ENGINE INNODB STATUS\G` → `LATEST DETECTED DEADLOCK`（只留最近一次）。兩邊各自睇 **HOLDS** / **WAITING**、**index 名**（`PRIMARY` vs `idx_user_id`）、`lock_mode`（X、`rec but not gap`、gap、insert intention）。
- `WE ROLL BACK TRANSACTION (n)`：InnoDB 犧牲 undo 較細嗰邊。
- 長期開 `innodb_print_all_deadlocks=ON` 寫 error log。8.0 用 `performance_schema.data_locks` / `data_lock_waits` 睇**當前**等待；死鎖一經回滾，現場可能已經散。

**點修**

- 雙方都改做**同一條加鎖路徑**：先按主鍵集合 `SELECT ... WHERE id IN (...) ORDER BY id FOR UPDATE`，再更新。跨帳戶就按 `account_id`／`id` **由細到大**排完先鎖。
- `WHERE` 必須走索引；事務內唔好做 HTTP／RPC；應用對 1213 **由事務開頭**有限次重試（已經成個 rollback）。
- 熱點行（手續費帳戶、熱門 uid）用排隊串行或分桶，減少並發搶同一組索引記錄。

### 追問／陷阱

- **改 RC 就唔會死鎖？** 唔會。RC 減少 gap lock，類型 (b) 會少；呢類「主鍵 vs 二級」**行鎖互等**照樣發生。治本仍然係固定加鎖順序 + 短事務 + 好索引。
- **重試只重跑最後一條 `UPDATE`**：錯。1213 已回滾成個事務，只重最後一句會漏咗前面已撤銷嘅改動。
- **關 `innodb_deadlock_detect`**：熱點下行死鎖檢測本身可以好食 CPU，有人會關並調細 `innodb_lock_wait_timeout` 變成 1205 兜底。呢個只係減輕檢測成本，唔係根治；監控要把 1213 同 1205 分開告警。
