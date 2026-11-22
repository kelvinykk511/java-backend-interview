# 大表 Online DDL（加列／改結構）同 MDL 卡死

## 面試題

千萬級線上訂單表要加一個可空列，點樣盡量唔影響寫入？如果出現大片 `Waiting for table metadata lock`，通常發生咗咩事？

## 標準答案

- **先問可不可以 INSTANT**。`ALGORITHM=INSTANT` 只改數據字典，唔重建表、唔複製行，秒級完成，期間可以繼續 DML。
  - 8.0.12 起支援 **表尾**加列，而且加列在可走 INSTANT 時會把它當預設算法。
  - 8.0.29 起多數情況可以喺**任意位置**加／刪列；每次 INSTANT 加／刪會產生一個 row version，最多 64 個（`INFORMATION_SCHEMA.INNODB_TABLES.TOTAL_ROW_VERSIONS`），到上限就要先 `INPLACE`／`COPY` 重建，計數先會歸零。
- **生產請寫明 `ALGORITHM=INSTANT`**。唔支援就立即報錯，你再改方案。如果唔指定算法，一旦 INSTANT 用唔到（見下方限制），MySQL 可能**默默改走 `INPLACE`／`COPY` 重建**，主庫 IO、從庫延遲一齊爆。
- **INSTANT 用唔到就唔好硬喺高峰直接 ALTER**。
  - `INPLACE`：視乎操作可能仍要重建表，耗 IO；即使 `LOCK=NONE` 准併發 DML，複製延遲同磁碟仍然要還。
  - `COPY`：建新表再拷數據，寫入影響最大，線上大表基本唔用。
  - 工具：忙表優先 **gh-ost**（跟 row binlog 灌影子表，可暫停／限速，唔使觸發器）；**pt-osc** 用觸發器同步變更，高峰寫會放大，未必「更穩」。兩者都要有主鍵／唯一鍵，**cut-over 仍然要短暫 MDL**，長事務一樣擋得住。
- **MDL 先於「快唔快」**：任何 DDL（包括 INSTANT）都要攞表嘅排他 MDL。前面有長事務／未提交查詢拎住共享 MDL，DDL 就停喺 `Waiting for table metadata lock`；之後新嚟嘅查詢、DML 要等呢把排他鎖，會**再排喺呢個 DDL 後面**，整張表看起來死咗。INSTANT 只係搶到鎖之後好快，**搶鎖之前一樣可以卡死業務**。
- **排查同操作順序**：
  1. `SHOW PROCESSLIST`、`information_schema.innodb_trx`、`performance_schema.metadata_locks`，搵「自己唔係 Waiting、但持住呢張表」嘅長事務（好多時係 `Sleep`／對帳沒 commit）。
  2. 結束或殺掉呢條事務，唔好先殺 DDL 本身（殺咗個等緊嘅 DDL，下一次仲會撞同一把鎖）。
  3. DDL 設合理 `lock_wait_timeout`，低峰執行；重建類操作要盯從庫延遲，必要時先喺從庫／預發演練。

## 常見追問／陷阱

- 「8.0 加列一定 INSTANT，唔會鎖」✗。以下情況 INSTANT 用唔到，唔寫 `ALGORITHM` 就可能重建：8.0.29 之前喺中間插列、`ROW_FORMAT=COMPRESSED`、表有 FULLTEXT、臨時表、同一次 ALTER 夾咗唔支援 INSTANT 嘅操作、已達 64 個 row version、加 **AUTO_INCREMENT** 列或改欄位類型等。就算真係 INSTANT，等 MDL 期間一樣會堵後面所有語句。
- 「`LOCK=NONE`／半同步就唔會卡、唔會有延遲」✗。`LOCK=NONE` 只描述重建期間可否 DML，唔取消 cut-over／開波嗰一下排他 MDL；半同步保證嘅係 binlog 到達從庫 relay，唔係 apply 完、更唔係冇 MDL。
- 「改用 pt-osc 就一定更穩」✗。觸發器令每個寫都多一次維護；busy 訂單表通常 gh-ost 更合適。工具代替唔到「先清長事務」。
- 從庫要重放同一套 DDL／重建，主庫看起來幾秒、從庫可能落後好多；讀寫分離環境要一齊睇延遲。

## 一句話總結

能 INSTANT 就顯式指定；否則用 gh-ost 等工具控速。DDL 卡住先查長事務同 MDL 排隊鏈，唔好假設「加列＝冇鎖」。
