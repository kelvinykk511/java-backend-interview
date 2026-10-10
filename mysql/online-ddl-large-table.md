# 大表加字段：Online DDL

**問題**：千萬級線上表要加一列，點做唔影響業務？

- MySQL 8.0：`ALGORITHM=INSTANT` 加列（尾部/8.0.29 後任意位置）只改元數據，秒級。
- `INPLACE`：唔鎖寫但要重建表，耗 IO、主從延遲。
- 工具：gh-ost（binlog 同步影子表、可暫停/限速）、pt-online-schema-change（觸發器）。
- **MDL 陷阱**：DDL 要拿 MDL 寫鎖，如果前面有長事務/慢查詢持有 MDL 讀鎖，DDL 會等，後面所有查詢排隊在 DDL 後 → 整表卡死。先查 `information_schema.innodb_trx` 殺長事務，設 `lock_wait_timeout`。
- 低峰執行、先在從庫/預發驗證、監控主從延遲。
