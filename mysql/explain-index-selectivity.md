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

## 關聯

- [B+ 與最左前綴](index-bplus-leftmost.md)
- [Gap / Next-Key Lock](gap-lock-next-key.md)
