# MySQL：EXPLAIN 與索引選擇性

## 面試問題

看 `EXPLAIN` 時你先看哪幾欄？什麼叫索引選擇性（selectivity）？為什麼「有建索引」查詢還是可能全表掃？

## 模範答案（面試可講）

- 先看：`type`（`const`／`eq_ref`／`ref`／`range`／`index`／`ALL`）、`key`（實際用哪個索引）、`rows`（估算掃描行數）、`Extra`（`Using where`／`Using index`／`Using filesort`／`Using temporary`）。
- **選擇性**：某欄（或組合）能區分出多少不同值；越接近「幾乎唯一」越適合當索引前綴。公式直覺：`distinct(col) / count(*)` 越高越好。
- 低選擇性欄（如性別、狀態只有幾個值）單獨建索引，優化器常覺得**掃索引不如掃表**，尤其結果集很大時。
- 還會放棄索引的情況：對索引列做函數／隱式型別轉換、模糊前綴 `%xxx`、統計資訊過舊、`OR` 難以合併、回表成本高於全表掃。
- 複合索引仍守**最左前綴**；覆蓋索引（`Using index`）可少回表，常比「有索引但每次回表」更快。

## 常見追問／陷阱

- `rows` 是**估算**不是精確值；慢查還要對 `SHOW PROFILE`／實際執行／慢查日誌。
- `type=index` 不是很好——代表**掃完整個索引**，常常接近全表成本。
- 「給每個 WHERE 欄都建單欄索引」未必好；優先**高選擇性 + 常見查詢形狀**的複合索引。

## 深入（2026-11-08）：`rows` 估算、選擇性同揀錯索引

### 面試問題

`EXPLAIN` 嘅 `rows` 係點估算出嚟？點解「有建索引」優化器仲會走全表掃，甚至揀咗選擇性更差嘅索引？

### 答法

- `rows` 係優化器**估算**「呢一步大概要讀幾多行」，**唔係**實際執行行數。主要來自 InnoDB 持久化統計（`mysql.innodb_table_stats` 嘅 `n_rows`，`mysql.innodb_index_stats` 嘅 `n_diff_pfx`——索引各前綴大概有幾多個不同值）。`SHOW INDEX` 嘅 `Cardinality` 就係呢個近似值。
- 單列等值查詢嘅直覺：`rows ≈ n_rows / n_diff_pfx`（每把鑰匙對應幾多行）。**選擇性** = `不同值數量 / 總行數`，越接近 1 越值得放聯合索引最左。
- 多個獨立條件若冇聯合索引／histogram，優化器常假設各條件獨立再把選擇性相乘，**相關欄位**（例如 `status` + `created_at`）會估錯。
- **值分佈嚴重不均**時，平均值會骗人：`status` 只有幾個值、`filled` 佔 95%，用「3 個不同值」去估會當每種 status 差唔多，實際掃 `filled` 接近全表。MySQL 8.0 可用 `ANALYZE TABLE t UPDATE HISTOGRAM ON status` 糾正。
- 有索引仲放棄／揀錯的常見原因：
  1. **低選擇性 + 大量回表**：`SELECT *` 走二級索引再逐行回表，成本可以高過順序掃主鍵（聚簇）。
  2. **統計過舊**：大批寫入後自動 recalc 未跟上，`ANALYZE TABLE` 刷新。
  3. 對索引列做函數、隱式型別轉換、左模糊 `%xxx`、違反最左前綴——索引用唔到，唔關選擇性事。
  4. `type=index` 係**掃晒成條二級索引**，成本常常接近 `ALL`，唔係「用到索引就好」。
- `filtered`：估計通過 WHERE 之後仲剩幾 % 行；傳去下一層（例如 join）大約係 `rows * filtered / 100`。
- 證實唔好只靠 `EXPLAIN`：MySQL 8.0.18+ 用 `EXPLAIN ANALYZE` 對比估算 vs 實際行數／耗時；再對 `SHOW INDEX`、`mysql.innodb_index_stats`。慢查仲要排除鎖等待同磁盤 IO。
- 修：按查詢形狀建**高選擇性開頭**嘅聯合索引（`user_id` 而唔係淨 `status`；常見係 `(user_id, status)`）；覆蓋索引減少回表；刷新統計；`FORCE INDEX` 只作短避險，唔係常態。

### 追問／陷阱

- `rows` 估得細但查詢好慢 → 統計錯、回表爆炸，或者根本係鎖等待／IO，唔好只信 EXPLAIN。
- 「Cardinality 高就一定會用呢個索引」唔成立——仲要計回表、是否覆蓋、同其他條件一齊嘅成本。
- 給每個 WHERE 欄各建一個單欄索引，低選擇性嗰個（狀態、刪除標記）反而可能誤導優化器。

## 關聯

- [B+ 與最左前綴](index-bplus-leftmost.md)
- [覆蓋索引](covering-index.md)
- [慢查詢排查](slow-query-troubleshooting.md)
- [Gap / Next-Key Lock](gap-lock-next-key.md)
