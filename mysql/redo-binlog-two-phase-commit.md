# redo log、binlog 與兩階段提交

## 面試問題

MySQL 已經有 binlog 了，為什麼 InnoDB 還要 redo log？兩者怎麼保證一致（兩階段提交）？

## 答法

**先分清兩本帳**

| | redo log | binlog |
|---|---|---|
| 誰寫 | InnoDB 引擎層 | MySQL Server 層（所有引擎） |
| 記什麼 | 物理：哪一頁改了什麼 | 邏輯：SQL / 行變更（ROW） |
| 用途 | **崩潰恢復**（crash-safe） | **主從複製、時間點恢復、CDC**（Canal/Debezium） |
| 寫法 | 固定大小循環寫 | 追加寫，滾動文件 |

**為何要 redo（WAL）**：改數據頁先改內存 Buffer Pool，順序寫 redo 就算提交成功；髒頁之後慢慢刷。掛了靠 redo 重放，避免每次提交都隨機寫數據頁。

**兩階段提交（內部 XA）**

1. 寫 redo，狀態 **prepare**。
2. 寫 binlog。
3. redo 標記 **commit**。

**崩潰恢復規則**：redo 是 prepare 時，去查 binlog 有沒有這個事務（XID）——有且完整就提交，沒有就回滾。這樣**主庫數據（redo）和從庫數據（binlog）不會分叉**。

**刷盤參數（「雙 1」）**

- `innodb_flush_log_at_trx_commit=1`：每次提交 redo fsync。
- `sync_binlog=1`：每次提交 binlog fsync。
- 調成 0/2 或 N 換吞吐，代價是 OS/機器掛時丟最近一段已「提交」的事務。資金類庫一般堅持雙 1。

## 常見追問 / 陷阱

「只寫 redo 不寫 binlog，或反過來，會怎樣？」

→ 先 redo 後 binlog、中間掛：主庫恢復有這筆，從庫/CDC 沒有 → **主從不一致**、下游漏消息。先 binlog 後 redo、中間掛：從庫有、主庫沒有。兩階段提交就是為了堵這個窗口。陷阱：把 **undo log**（回滾＋MVCC）和 redo 混為一談——undo 是「改回去」，redo 是「重做」。

## 小練習

交易所用 Canal 訂閱 binlog 推送入帳事件。DBA 為了壓測把 `sync_binlog` 改成 1000，機器斷電後發生什麼？為什麼下游可能「少一筆入帳通知」而主庫餘額卻是對的（或反過來）？
