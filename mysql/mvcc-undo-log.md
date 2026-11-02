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

---

## 深入（2026-11-02）：ReadView 可見性規則 + RC vs RR

**面試問題**：InnoDB 點樣判斷一行數據對當前事務「睇唔睇得到」？RC 同 RR 差喺邊？

**答法**

- 每行有隱藏列 `DB_TRX_ID`（最後改佢嘅事務 id）同 `DB_ROLL_PTR`（指去 undo log 舊版本），舊版本串成版本鏈。
- 快照讀（普通 `SELECT`）會攞一個 **ReadView**，入面有：
  - `m_ids`：建 ReadView 嗰刻仲活躍（未提交）嘅事務 id 列表
  - `min_trx_id`：m_ids 最細值
  - `max_trx_id`：下一個會分配嘅事務 id
  - `creator_trx_id`：自己
- 判斷某版本 `trx_id`：
  1. `== creator_trx_id` → 自己改嘅，睇得到
  2. `< min_trx_id` → 建 ReadView 前已提交，睇得到
  3. `>= max_trx_id` → 建 ReadView 之後先開嘅事務，睇唔到
  4. 喺中間：喺 `m_ids` 入面 → 未提交，睇唔到；唔喺 → 已提交，睇得到
  - 睇唔到就沿 `DB_ROLL_PTR` 搵上一個版本再判斷。
- **RC**：每條 SELECT 都新建 ReadView → 會見到別人新提交嘅數據（不可重複讀）。
- **RR**：事務內**第一條快照讀**先建 ReadView，之後一直用同一個 → 可重複讀。注意唔係 `BEGIN` 嗰刻建（要即刻建用 `START TRANSACTION WITH CONSISTENT SNAPSHOT`）。

**追問／陷阱**

- **快照讀 vs 當前讀**：`UPDATE/DELETE/SELECT ... FOR UPDATE` 係當前讀，讀最新已提交版本並加鎖，唔睇 ReadView。
- RR 經典怪事：A 先 `SELECT` 見 10 行；B 插 1 行並提交；A `UPDATE ... WHERE 同條件` 返回 affected 11；A 再 `SELECT` 見 11 行。因為 UPDATE 係當前讀改咗 B 嗰行，嗰行 `trx_id` 變成 A 自己 → 規則 1 睇得到。所以 MVCC 只係防快照讀幻讀，要完全防就用 `FOR UPDATE`（next-key lock）。
- undo 舊版本唔可以即刻刪：purge 線程要等**冇任何 ReadView 仲需要佢**先清 → 長事務會拖住 purge。
