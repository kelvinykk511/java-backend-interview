# MySQL 慢查詢排查

**問題：** 線上某接口突然變慢，懷疑係 SQL，你點排查？

## 答案要點
- 開 slow log（`long_query_time`、`log_queries_not_using_indexes`），用 `pt-query-digest` 按總耗時排序，先搞最痛嗰條。
- `EXPLAIN` 睇 type（ALL/index 危險）、key、rows、Extra（Using filesort / Using temporary）。
- 常見原因：冇索引／索引失效（函數、隱式轉換、左模糊、OR、違反最左前綴）、回表多、深分頁、大事務鎖等待、統計信息過期選錯索引。
- 唔係 SQL 本身慢：`SHOW PROCESSLIST` / `information_schema.innodb_trx` 睇鎖等待、MDL；buffer pool 命中率、磁盤 IO。
- 修：加/改聯合索引、覆蓋索引、改寫 SQL、`ANALYZE TABLE`、必要時 `FORCE INDEX`（短期）。

## 追問／陷阱
- 「EXPLAIN 用到索引點解仲慢？」→ rows 大、回表多、選擇性差、或者其實係鎖等待。
- `varchar` 欄位用數字查 → 隱式轉換令索引失效。
