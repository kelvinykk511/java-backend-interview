# 深分頁：LIMIT 大 offset 為什麼慢

## 面試問題

`SELECT * FROM orders WHERE user_id = ? ORDER BY id DESC LIMIT 1000000, 20` 為什麼慢？怎麼改？

## 答法

- **原因**：MySQL 要先找出 offset + 20 行，再把前 100 萬行丟掉。走二級索引時，每一行還要**回表**取 `SELECT *` 的欄位 → 100 萬次回表，幾乎全白做。
- **改法 1：延遲關聯（deferred join）**
  ```sql
  SELECT o.* FROM orders o
  JOIN (SELECT id FROM orders WHERE user_id = ?
        ORDER BY id DESC LIMIT 1000000, 20) t ON o.id = t.id;
  ```
  子查詢只在索引上掃（`(user_id)` 二級索引葉子本身就帶主鍵 id，覆蓋索引），不回表，最後只回表 20 行。仍然要掃 100 萬條索引記錄，只是便宜很多。
- **改法 2：游標分頁（keyset / seek）** — 最推薦
  ```sql
  SELECT * FROM orders WHERE user_id = ? AND id < :lastId
  ORDER BY id DESC LIMIT 20;
  ```
  直接從索引定位，只讀 20 行，跟第幾頁無關。代價：不能跳到第 N 頁，只能「下一頁」。很多交易所歷史成交 API 用 `fromId` 就是這個思路。
- **產品層**：限制最大可翻頁數；大批量導出走異步任務、從庫或數倉，不在線上庫翻頁。

## 常見追問／陷阱

- **排序欄位不唯一**（例如 `created_at`）：同一時間戳多行，用 `created_at < ?` 會漏或重複。要帶上唯一欄位做組合條件：
  `created_at < ? OR (created_at = ? AND id < ?)`，配 `(created_at, id)` 索引。
- **翻頁期間有新數據插入**：offset 分頁會出現重複或漏行；keyset 按值定位，穩定得多。
- `COUNT(*)` 總頁數本身也貴：大表可以不顯示精確總數，或用估算／單獨計數表。
