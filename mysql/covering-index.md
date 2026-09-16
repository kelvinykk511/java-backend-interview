# MySQL：覆蓋索引（Covering Index）

## 面試問題

什麼叫覆蓋索引？為什麼有時「索引都命中了」還是慢，而加上覆蓋索引後會快很多？

## 答法

- **覆蓋索引**：查詢需要的**所有欄位**都在某個索引裡，InnoDB 用二級索引就能回傳結果，**不必回表**讀聚簇索引（主鍵那棵 B+ 樹）。
- **EXPLAINextra**：常出現 `Using index`（用到覆蓋）；若還有 `Using where` 是在索引上過濾。
- **為什麼快**：少一次（或大量）隨機 I/O 回表；高 QPS 列表／計數場景收益明顯。
- **代價**：索引更寬 → 佔空間、寫入更慢；欄位常變就別硬塞進覆蓋索引。
- **設計口訣**：`WHERE` / `ORDER BY` / `GROUP BY` 用到的列放前面（最左前綴），`SELECT` 多出來的列可當「include」尾巴（聯合索引多帶幾列）。
- **面試一句話**：覆蓋 = 索引裡已經有 SELECT 要的列，省掉回表；用 `Using index` 驗證。

## 常見追問 / 陷阱

「主鍵索引算覆蓋嗎？」

→ 聚簇索引本身就帶整行，談覆蓋通常是針對**二級索引**能否免回表。另：`SELECT *` 很難覆蓋；只查必要欄位才容易吃到覆蓋索引。

## 小練習

表有 `(status, created_at, id)`，查 `SELECT id, status FROM t WHERE status=1 ORDER BY created_at LIMIT 20`。現有索引 `(status, created_at)` 算不算覆蓋？要不要改成 `(status, created_at, id)`？為什麼？
