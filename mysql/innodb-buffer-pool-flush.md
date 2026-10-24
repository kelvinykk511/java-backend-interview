# InnoDB Buffer Pool 與刷髒頁

## 面試題
一條 UPDATE 改完點解唔即刻寫數據文件？Buffer Pool 點運作？乜嘢時候刷髒頁？點解 MySQL 會偶發「抖一下」（TPS 突然跌）？

## 答案要點
- **Buffer Pool**：InnoDB 喺內存緩存 16KB page。讀寫都先打 Buffer Pool；改咗嘅 page 叫**髒頁**。持久性靠 **redo log（WAL）**：先寫 redo 就算提交成功，髒頁之後異步刷盤。
- **改良 LRU**：分 young / old 兩段（old 預設約 3/8，`innodb_old_blocks_pct=37`）。新讀入 page 插 old 頭；喺 old 停留超過 `innodb_old_blocks_time`（1000ms）再被訪問先升 young → 防全表掃描/mysqldump 洗走熱數據。
- **四個刷髒時機**：
  1. **redo log 寫滿**：checkpoint 推唔前，要同步刷髒，**寫入全停** → 最傷，抖動主因。
  2. **free page 唔夠**：淘汰髒頁要先刷，查詢線程自己等刷盤 → 查詢變慢。
  3. **後台 page cleaner** 定期/自適應刷（adaptive flushing）。
  4. **正常關機**。
- **調優**：`innodb_io_capacity` 設到接近磁碟真實 IOPS（預設 200 對 SSD 太低 → 平時刷太慢，一滿就爆）；`innodb_max_dirty_pages_pct`；redo 開大（8.0.30+ `innodb_redo_log_capacity`）；SSD 用 `innodb_flush_neighbors=0`（8.0 預設 0）。
- **大小**：專用機約 50–75% 內存；`innodb_buffer_pool_instances` 減鎖競爭；命中率 ≈ 1 − `Innodb_buffer_pool_reads` / `Innodb_buffer_pool_read_requests`。
- **Change Buffer**：**非唯一**二級索引嘅修改，如果 page 唔喺內存，先記喺 change buffer，之後讀到/後台再 merge → 省隨機讀。**唯一索引用唔到**（要讀 page 檢查唯一性）。

## 追問 / 陷阱
- 「唯一索引 vs 普通索引寫入邊個快？」→ 寫多讀少（日誌、流水）普通索引有 change buffer 更快；但寫完即刻讀嘅場景 merge 頻繁，change buffer 反而白做。業務要唯一就用唯一索引，唔好為性能犧牲正確性。
- 「命中率 99% 就冇問題？」→ 唔一定；全表掃描/大查詢會洗 LRU，要睇 `Innodb_buffer_pool_wait_free`、髒頁比例同 checkpoint age。
