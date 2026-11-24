# MySQL：覆蓋索引（Covering Index）

## 面試問題

什麼叫覆蓋索引？`Using index` 跟 `Using index condition` 差在哪？為什麼 `SELECT *` 常常吃不到覆蓋？

## 答法

**是什麼**

- **覆蓋索引**：查詢需要的**所有欄位**（`SELECT` + `WHERE` / `ORDER BY` / `GROUP BY` 用到的）都在某個索引裡 → InnoDB 只掃二級索引就能回傳，**不必回表**讀聚簇索引。
- EXPLAIN `Extra` 出現 **`Using index`**＝這次是覆蓋（索引內搞定）。
- **對比**：一般二級索引葉子只有「索引列 + 主鍵」→ 還要拿 PK 回聚簇索引取其餘列；覆蓋＝省掉這步隨機 I/O。

**什麼時候有用**

- `SELECT` 只取索引裡已有的列（列表、計數、存在性判斷）。
- 高 QPS、回表比例高時收益最明顯；用 `EXPLAIN` 對照加索引前後是否從「索引 + 回表」變成 `Using index`。

**`Using index` vs `Using index condition`（ICP）**

| Extra | 意思 |
|-------|------|
| `Using index` | **覆蓋**：結果欄都在索引，不回表 |
| `Using index condition` | **Index Condition Pushdown**：把部分 WHERE 下推到引擎，在索引層先過濾，**仍可能回表**拿完整行 |

別把 ICP 講成覆蓋；兩者可同時出現，但語意不同。

**代價與約束**

- 索引更寬 → 佔空間、寫入／更新變慢；常變欄位別硬塞進覆蓋尾巴。
- 仍守**最左前綴**：聯合索引從左連續匹配；範圍條件會截斷後續列的有序利用。
- 設計口訣：過濾／排序列靠左，多出來的 `SELECT` 列當尾巴「帶上」。

**面試一句話**：覆蓋 = 索引裡已有查詢要的列，省回表；用 `Using index` 驗證，別跟 ICP 的 `Using index condition` 搞混。

## 常見追問 / 陷阱

「主鍵索引算覆蓋嗎？」

→ 聚簇索引葉子本來就整行，談覆蓋通常針對**二級索引能否免回表**。

「`SELECT *` 呢？」

→ 幾乎一定要回表（除非極窄表、列全在索引），**直接殺死覆蓋**；只查必要欄位才容易吃到。

「範圍條件打在非最左／中間列？」

→ 最左前綴用不上或被截斷 → 優化器可能放棄該索引，覆蓋設計也白做。等值在前、範圍／排序在後。

## 小練習

表有索引 `(status, created_at)`，查 `SELECT id, status FROM t WHERE status=1 ORDER BY created_at LIMIT 20`。算不算覆蓋？要不要改成 `(status, created_at, id)`？為什麼？


## 深入：InnoDB 隱式主鍵同「加多一列先算覆蓋」（2026-11-24 warm-up）

**面試題**：表主鍵係 `id`，已有 `(user_id, created_at)`。`SELECT id, user_id, created_at FROM orders WHERE user_id=? ORDER BY created_at LIMIT 20` 算唔算覆蓋？要唔要再加一列 `id`？同 `Using index condition` 點分？

- **InnoDB 二級索引葉子已經帶主鍵**。`(user_id, created_at)` 實際等同 `(user_id, created_at, id)`（PK 在尾）。上面呢條查詢需要嘅列都在索引裡，而且 `user_id` 等值 + `created_at` 有序，通常已經係 **`Using index`**，**唔使再把 `id` 加一次**。
- 若 `SELECT` 多要 `amount` 等未包含嘅列，先會回表；呢時先考慮把穩定、常被讀、少更新嘅列加喺索引尾巴。`SELECT *` 幾乎一定回表，覆蓋直接被殺死。
- **唔好為咗「帶上 id」亂建 `(user_id, id, amount)`**。最左前綴變成 `user_id, id`，對 `WHERE user_id=? AND status=?` 或 `ORDER BY created_at` 用唔到後續列，優化器可能根本唔揀呢條索引。
- **前綴索引不能覆蓋該列**：`INDEX(user_id, created_at(10))` 只存前綴，唔能單靠索引交出完整 `created_at`，就算 EXPLAIN 用咗呢條索引都要回表。
- 再分一次：**`Using index` = 覆蓋、唔回表**；**`Using index condition` = ICP**，索引層先過濾，**仍然可能回表**。兩者可以同時出現，但語意唔同。
- 用 `EXPLAIN`（必要時 `EXPLAIN ANALYZE`）驗證，唔好假設「建咗覆蓋索引就一定用」。選擇性差、統計過期、或排序／分組用唔上最左前綴，優化器會放棄。索引越寬，寫入同緩衝池壓力越大，只為偶發查詢加很寬嘅覆蓋通常唔划算。

### 追問

- 「主鍵索引本身算唔算覆蓋？」聚簇索引葉子已經係整行，面試講覆蓋通常係指**二級索引能否免回表**。
- `COUNT(*)` 有時可以只掃一條較窄嘅二級索引（依然係 index-only），呢個係覆蓋思維，但優化器亦可能揀另一條更細嘅索引。
