# redo log、binlog 與兩階段提交

## 面試問題

MySQL 已經有 binlog，點解 InnoDB 仲要 redo log？崩潰時點樣靠兩階段提交，令主庫數據同 binlog（從庫／Canal）一致？「雙 1」同 group commit 你會點講？

## 答法

**先分清兩本帳**

| | redo log | binlog |
|---|---|---|
| 誰寫 | InnoDB 引擎層 | MySQL Server 層（所有引擎） |
| 記什麼 | 物理：邊一頁改咗咩 | 邏輯：SQL／行變更（通常 ROW） |
| 用途 | **崩潰恢復**（crash-safe） | **主從複製、時間點恢復、CDC**（Canal／Debezium） |
| 寫法 | 固定大小循環寫 | 追加寫，滾動文件 |

**點解要 redo（WAL）**：改數據頁先改內存 Buffer Pool，提交時順序寫 redo 就算數；髒頁之後先慢慢刷。掛咗靠 redo 重放，避免每次提交都隨機寫數據頁。binlog 負責「把變更告訴別人」，救唔到 InnoDB 自己嘅崩潰恢復。

**兩階段提交（內部 XA，唔係業務嗰套分布式 XA）**

1. 寫 redo，狀態 **prepare**（通常會落盤，視刷盤參數）。
2. 寫 binlog（完整事件帶 XID）。
3. redo 標記 **commit**。

**崩潰恢復規則**：redo 停留喺 prepare 時，去 binlog 搵呢個事務嘅 XID——**有且完整就提交，冇就回滾**。咁樣主庫數據（redo）同從庫／CDC（binlog）先唔會分叉。

- 只有 prepare、binlog 未寫完 → 回滾（外界當呢筆冇發生過）。
- binlog 已完整、redo 未打 commit 旗 → 恢復時補提交。
- 兩邊都完成 → 正常已提交。

**刷盤參數（「雙 1」）**

- `innodb_flush_log_at_trx_commit=1`：每次提交（含 prepare）redo `fsync`。
- `sync_binlog=1`：每次提交 binlog `fsync`。
- 調成 0／2 或 N 換吞吐，代價係**機器斷電**時可能丟最近一段已「提交」嘅事務，或者主庫同 binlog 分叉。資金類庫一般堅持雙 1。

參數含義（重點分清「進程崩潰」同「機器斷電」）：

| 參數 | 行為 | `kill -9`／進程崩潰（OS cache 仲在） | 拔電／kernel panic |
|------|------|----------------------------------------|---------------------|
| `flush_log=1` + `sync_binlog=1` | 每次都 fsync | 一致、唔丟 | 一致、唔丟 |
| `flush_log=2` | 每次寫入 OS cache，約每秒 fsync | 多數唔丟（已 write） | 可能丟約 1 秒 redo |
| `flush_log=0` | redo 緩衝約每秒先 write＋fsync | 可能丟約 1 秒 | 可能丟約 1 秒 |
| `sync_binlog=N`（N>1） | 累積 N 次先 fsync binlog | 多數唔丟 | binlog 尾部可能冇咗 |

**只改一邊嘅典型分叉（斷電）**

- `flush_log=1`、`sync_binlog=N`：redo 已提交，binlog 尾部未落盤 → **主庫有呢筆，從庫／Canal 冇**（餘額加咗，入帳通知少一筆）。
- `flush_log=2`、`sync_binlog=1`：binlog 已落盤，redo 可能未 fsync → **從庫／Canal 有，主庫恢復後冇**（下游通知到，主庫餘額冇加）。雙 1 就係為咗堵呢兩個窗口。

**Group commit**：雙 1 唔等於「一筆事務一次獨立 fsync」。InnoDB redo 同 binlog 都可以把同一時間窗口內多個事務**合併一次 fsync**（binlog group commit），所以堅持雙 1 仍然可以有像樣嘅吞吐。資金庫唔好為咗 TPS 隨便關雙 1，先睇磁碟同 group commit 係咪已經夠。

## 常見追問 / 陷阱

「只寫 redo 唔寫 binlog，或反過來，會點？」

→ 兩階段提交就係為咗堵呢個窗口。陷阱：

- 把 **undo log**（回滾＋MVCC）同 redo 混為一談——undo 係「改返轉頭」，redo 係「重做」。
- 以為內部 2PC = 業務分布式 XA／TCC。呢度只係**同一台 MySQL 入面** redo 同 binlog 對齊。
- 以為開咗 **semi-sync** 就可以關雙 1。Semi-sync 係等至少一個從庫 ack binlog，降低「主庫已提交、從庫未收到」喺主掛切從時嘅丟失；**替代唔到**本機 redo／binlog 落盤。資金庫兩者都要。
- 以為 `kill -9` 同拔電一樣。進程崩潰時 OS cache 通常會被刷落盤，`flush_log=2` 多數仲在；拔電先會暴露「冇 fsync」嘅丟數同分叉。

## 小練習

見 `sessions/` 當日 drill。舊題：`sync_binlog=1000` 機器斷電後，Canal 下游同主庫可能點樣不一致？
