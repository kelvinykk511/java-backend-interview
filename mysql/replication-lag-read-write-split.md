# MySQL 主從延遲 & 讀寫分離「讀唔到自己啱啱寫」

## 面試題
做咗讀寫分離之後，用戶啱啱下單就查唔到訂單。點解？點排查、點解決？

## 標準答案
- **複製流程**：主庫 commit 寫 binlog → 從庫 IO 線程拉去 relay log → SQL／applier 線程重放。預設**異步複製**，主庫唔等從庫，所以寫完即刻讀從庫，好可能未 apply。
- **睇延遲**：`SHOW REPLICA STATUS` 嘅 `Seconds_Behind_Source`（粗略，IO 線程斷咗或大事務時會誤導）；更準：pt-heartbeat 心跳表、比較主從 GTID 執行集。
- **常見原因**：
  - 大事務：主庫一次 `DELETE` 百萬行，從庫都要同樣耐重放。
  - 大表 DDL（從庫要再做一次）。
  - 從庫 apply 唔夠並行：開 `replica_parallel_workers` + `replica_parallel_type=LOGICAL_CLOCK`，配合 WRITESET 依賴追蹤。
  - 從庫機器差、或者跑報表／慢查詢搶資源。
  - 無主鍵表 + row 格式，從庫回放要掃全表。
- **解決「讀己之寫」**（由簡單到精細）：
  1. 寫後關鍵讀強制走主庫：同一請求寫後讀走主；或者寫後 N 秒內該用戶走主（Redis／cookie 打標記）。
  2. GTID 等待：寫完拿 GTID，讀從庫前 `WAIT_FOR_EXECUTED_GTID_SET(gtid, timeout)`，超時就 fallback 主庫。
  3. 業務層避開：下單接口直接返回訂單數據，唔使即刻再查；MQ 消息帶齊下游需要嘅數據。
  4. 延遲超閾值自動摘除從庫或切主（小心壓垮主庫）。
- **治本**：拆大事務分批（每批幾千行 + sleep）、大 DDL 用 gh-ost／pt-osc 控速、開並行複製、監控延遲告警。

## 常見追問／陷阱
- 「改半同步就冇延遲」✗：半同步（AFTER_SYNC）只保證 binlog 已到從庫 relay log，**唔保證已 apply**；而且等超過 `rpl_semi_sync_source_timeout` 會自動退化成異步。半同步解決嘅係「主庫掛咗唔丟數據」，唔係讀延遲。
- 事務內讀寫分離：同一個 `@Transactional` 入面嘅讀一定要走主庫，否則讀唔到自己未提交（或啱提交）嘅寫；動態數據源／ShardingSphere 要用 hint 或者「事務內強制主庫」規則。
- 「查唔到」唔一定係延遲：亦可能係消息喺事務 commit 前已發出（見 Spring `@Transactional` 內發 MQ）。

## 一句話總結
異步複製必有延遲；關鍵讀走主或 GTID 等待，治本靠拆大事務 + 並行複製。
