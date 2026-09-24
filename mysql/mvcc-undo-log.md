# MySQL InnoDB：MVCC 與 undo log

## 面試問題

InnoDB 的 MVCC 是怎麼讓「讀」不太擋「寫」的？undo log 在裡面扮演什麼角色？

## 答法

- **MVCC**：多版本並發控制。每行帶隱藏欄位（大致：`DB_TRX_ID`、`DB_ROLL_PTR`），更新不直接抹掉舊值，而是留下可回溯的版本鏈。
- **Read View**：一致性讀（普通 `SELECT`）建立時產生「誰可見」的快照：哪些交易已提交可見、哪些未提交／之後才開始的不可見。
  - **RC**：每條語句可能拿新的 Read View（能讀到其他交易已提交的新版本）。
  - **RR**：通常整個事務共用第一次的 Read View（快照讀較穩）。
- **undo log**：舊版本（或回滾資訊）在 undo；`ROLL_PTR` 指向上一版。可見性判斷沿著版本鏈找「對這個 Read View 可見」的那一版。
- **跟鎖的關係**：快照讀走 MVCC，通常不加行鎖；當前讀（`FOR UPDATE` / `UPDATE` / `DELETE`）要最新版並加鎖，才可能碰到 Gap / Next-Key。
- **回滾與 purge**：事務回滾靠 undo；不再需要的舊版由 purge 清理。別跟 **redo** 搞混：redo 保耐久（怎麼重放），undo 保回滾與舊版本。

## 常見追問／陷阱

「RR 下為什麼還可能幻讀？」→ 快照讀靠 MVCC 能避免多數幻讀；當前讀要靠 Gap／Next-Key。別把「MVCC = 完全沒鎖」講死。

「長事務有什麼壞處？」→ Read View 持有太久，舊版 undo 清不掉，undo 膨脹、歷史鏈變長，查詢變慢。
