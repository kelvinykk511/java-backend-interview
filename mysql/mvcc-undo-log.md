# MySQL InnoDB：MVCC 與 undo log

## 面試題

InnoDB 的 MVCC 是怎麼讓「讀」不太擋「寫」的？undo log 在裡面扮演什麼角色？

## 模範答案（面試官向）

- **MVCC**：多版本並發控制。每行帶隱藏欄位（大致：`DB_TRX_ID`、`DB_ROLL_PTR` 等），更新不直接蓋死舊值，而是留下可回溯的版本鏈。
- **Read View**：一致性讀（如 RR 下的普通 SELECT）建立時產生「誰可見」的快照視圖：哪些交易已提交可見、哪些未提交／之後才開始的不可見。
- **undo log**：舊版本資料（或回滾所需資訊）存在 undo；`ROLL_PTR` 指向上一版。可見性判斷沿著版本鏈找「對這個 Read View 可見」的那一版。
- **跟鎖的關係**：快照讀走 MVCC，通常不加行鎖；當前讀（`SELECT ... FOR UPDATE` / `UPDATE` / `DELETE`）要最新版並加鎖，才可能碰到 Gap / Next-Key。
- **回滾與崩潰恢復**：事務回滾靠 undo；undo 也服務於 MVCC 版本鏈（purge 執行緒稍後清理不再需要的舊版）。

## 常見追問／陷阱

「RR 下為什麼還可能幻讀？」→ 快照讀靠 MVCC 能避免多數幻讀；當前讀要靠 Gap/Next-Key。別把「MVCC = 完全沒鎖」講死。另：不要把 undo 跟 redo 搞混——redo 保耐久（怎麼重放修改），undo 保回滾與舊版本。
