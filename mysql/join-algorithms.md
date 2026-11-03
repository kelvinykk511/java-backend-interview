# MySQL JOIN 算法：NLJ / BNL / Hash Join + 小表驅動大表

## 面試問題

`SELECT ... FROM orders o JOIN users u ON o.user_id = u.id` 喺 MySQL 入面係點執行？點解成日話「小表驅動大表」？被驅動表連接字段冇索引會點？

## 答法

- **驅動表 vs 被驅動表**：外層 loop 嗰張叫驅動表，每一行拎去內層表搵匹配。`EXPLAIN` 第一行通常就係驅動表。
- **Index Nested-Loop Join（NLJ）**：被驅動表連接字段**有索引** → 每行驅動表做一次 B+ 樹查找。成本約 `N × log M`，N 越細越好 → 所以「小表驅動大表」。
- **被驅動表冇索引**：
  - **8.0.18 之前：Block Nested-Loop（BNL）**。將驅動表一批行塞入 `join_buffer`，再全表掃被驅動表逐行對比。比較次數仍然係 `N × M`，只係減少掃表次數。`Extra: Using join buffer (Block Nested Loop)`。
  - **8.0.18+：Hash Join**（8.0.20 起完全取代 BNL）。細嗰邊建 hash 表（build），大嗰邊掃一次去 probe，等值連接近似 `N + M`。`Extra: Using join buffer (hash join)`。buffer 唔夠會落盤分塊。
- **「小表」要點理解**：唔係睇整張表大細，係睇**過濾條件之後參與 join 嘅行數 × 需要嘅列**。WHERE 過濾完得 100 行嘅大表都可以係「小表」。
- **BKA / MRR**：被驅動表用二級索引查回表時，MRR 將主鍵排序再回表，順序 I/O 更快；BKA 係 NLJ + MRR。

## 常見追問 / 陷阱

- **`LEFT JOIN` 唔可以隨便換驅動表**：左表一定係驅動表；但如果 WHERE 對右表字段做非 NULL 過濾（例如 `WHERE u.status=1`），優化器會改寫成 INNER JOIN，可以重新揀順序。
- **連接字段類型 / 字符集唔一致**（例如 `varchar` vs `int`、`utf8` vs `utf8mb4`）→ 隱式轉換令被驅動表索引失效，NLJ 退化成 hash join / 全表掃。線上好常見。
- **`STRAIGHT_JOIN`** 可以強制左表做驅動表，但係止痛藥；先修統計信息（`ANALYZE TABLE`）同索引。
- 大廠規範常寫「禁止超過 3 表 join」：原因係執行計劃難預測 + 難分庫分表，唔係 MySQL join 一定慢。CEX 場景：分庫之後跨庫 join 做唔到，通常改為應用層兩次查詢或寬表 / 冗餘字段。

關聯：[EXPLAIN 與索引選擇性](explain-index-selectivity.md)、[慢查詢排查](slow-query-troubleshooting.md)
