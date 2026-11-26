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


## 深入：半同步、SBS=0、事務內拆讀（2026-11-26 warm-up）

**面試題**：下單寫主庫成功，接口隨即查「我的訂單」卻係空；有人改半同步，有人把寫同讀放進同一個 `@Transactional` 但 SELECT 被中間件打去從庫。`Seconds_Behind_Source` 經常係 0。點解釋、點查、點改？

- **寫後即讀從庫，本來就可以係空**。主庫 commit 只保證自己嘅 binlog／redo；異步複製下，從庫 IO 拉 relay、applier 重放都係之後嘅事。前端 200ms 內刷新列表打從庫，係最典型嘅「讀唔到自己啱啱寫」。
- **半同步唔解決讀延遲**。`AFTER_SYNC` 只保證至少一個從庫已把事件刷入 **relay log** 先向客戶端提交，**唔保證 SQL／worker 已 apply**。超時（`rpl_semi_sync_source_timeout`）仲會自動退化成異步。半同步嘅目標係「主掛咗，已提交嘅數據唔丟」，唔係 session 一致性。
- **事務內把 SELECT 拆去從庫更糟**。同一個 `@Transactional` 入面自己啱寫嘅行，從庫既睇唔到未提交嘅寫，亦可能未 apply 啱提交嘅寫 → 一定空，甚至讀到舊狀態亂判斷。動態數據源／中間件要「事務內強制主庫」，唔可以按「SELECT 就走從」做語句級路由。
- **`Seconds_Behind_Source=0` ≠ 已追平**。數字靠從庫時鐘同「最後一條已執行事件」嘅時間戳估算：主從時鐘偏差可以顯示 0；大事務重放期間數字會失真，完咗先突然跳高；並行複製（`replica_parallel_workers`）下更唔準；IO／SQL 線程停咗時可能係 `NULL`，同 0 要分開。應用 **pt-heartbeat** 心跳表，或比較主從 GTID executed set。
- **改讀路徑**（由快到穩）：
  1. 寫後關鍵讀走主：同一次請求內；或該用戶寫後 N 秒 sticky（cookie／Redis 標記）。
  2. 寫完拿 GTID，讀從庫前 `WAIT_FOR_EXECUTED_GTID_SET(gtid, timeout)`，超時 fallback 主庫。
  3. 下單／出金接口直接返回已寫入嘅訂單數據，唔好先 commit 再立刻再查一次。
  4. 延遲超閾值摘除交易用從庫（要有上限，以免全部打爆主庫）。
- **報表／對帳** 嘅大查詢要隔離到**專用從庫**，唔好同交易讀共用一條 replica，否則 apply 被慢查拖住，所有「寫後讀從庫」一齊出事。
- **正確性以主庫為準**。餘額、凍結、訂單狀態呢類資金讀，寧願走主或等 GTID，唔好比從庫舊值做決策。

### 追問

- 「開並行複製就冇延遲？」只係加快 apply，大事務、DDL、從庫資源被報表打滿仍然會滯後。
- 「查唔到」唔一定係複製：亦可能係事務未 commit 就發咗查詢／MQ（見 Outbox）。兩邊要一齊排。
