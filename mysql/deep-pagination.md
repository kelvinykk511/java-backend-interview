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

---

## 深入（2026-11-14）：延遲關聯只省回表，深頁仍然線性慢

### 面試可以點追

`SELECT * FROM trades WHERE user_id=? ORDER BY id DESC LIMIT 500000, 20` 點解越翻越慢？延遲關聯夠唔夠？同游標分頁點揀？

### 答法

**OFFSET 唔係跳過**：InnoDB 仍然要沿索引走完 **offset + limit** 行，再把前面丟掉。`SELECT *` 若走二級索引，呢 50 萬行仲要逐行回表，幾乎全白做。第 1 頁快、第 2 萬頁超時，係同一條 SQL 的預期行為，唔係偶發慢查詢。

**延遲關聯只解決回表，解決唔到「走完 offset」**

- 子查詢只在覆蓋索引上拿 id（`(user_id, id)`，二級索引葉子本身帶主鍵），最後只回表 20 行。
- 深頁仍然要掃 50 萬條索引記錄，成本由「50 萬次回表」降成「50 萬次索引掃」，仍然隨頁碼線性變差，只係平好多，唔係 O(1)。
- 子查詢 `ORDER BY` 唔在索引上，仲會 filesort／臨時表，延遲關聯幾乎冇賺。

**游標／keyset 先係深翻頁正解**

```sql
SELECT * FROM trades
WHERE user_id = ? AND id < :lastId
ORDER BY id DESC
LIMIT 20;
```

用 `(user_id, id)` 直接 seek，只讀 20 行，同第幾頁無關。代價：不能隨便跳到第 N 頁。成交歷史 API 用 `fromId`／`cursor` 就係呢個。後台如果一定要跳頁，應限制最大頁，或把跳頁／導出放到從庫、異步任務或數倉，唔好同線上熱查共用 OFFSET 循環。

**對帳／批量掃表唔好用 `LIMIT offset, n` 循環**：offset 愈大愈慢，而且會重複讀已經掃過嘅索引前綴，拖高 CPU 同 buffer pool。改成 `WHERE id > :lastId ORDER BY id LIMIT n`，每次用上一批最後一個 id。

### 追問／陷阱

- **排序欄位唔唯一**（`trade_time`／`created_at`）：只寫 `trade_time < ?` 會漏或重複。正確係組合條件，並配組合索引：
  `WHERE user_id=? AND (trade_time < ? OR (trade_time=? AND id < ?)) ORDER BY trade_time DESC, id DESC`，索引 `(user_id, trade_time, id)`。MySQL 行構造器 `(trade_time, id) < (?, ?)` 同樣要核實有無用對應索引。
- **翻頁期間有插入／刪除**：OFFSET 靠「跳過 N 行」，資料一變就重／漏；keyset 按鍵值定位，穩得多。
- **精確 `COUNT(*)` 總頁**本身都可以好貴（就算有 `(user_id, symbol)` 都可能掃大量二級索引）。大表用估算、緩存計數或不顯示總頁；唔好每頁都打一次 COUNT。
