# Redis ZSet（有序集合）

## 面試問題

Redis `ZSet` 適合解決什麼問題？和 `List` / `Hash` 比，什麼時候該選它？

## 答法

- **結構**：member + score；按 score 排序，member 唯一。底層常見 **skiplist + hash**（既能按分排序，又能 O(1) 查某個 member 的分）。
- **典型場景**：排行榜、延遲隊列（score=執行時間戳）、熱門文章、滑動窗口限流計數、帶權重的優先隊列。
- **對比**：
  - `List`：只關心進出順序，不按分數排名。
  - `Hash`：field→value，沒有天然「按分數排序取 TopN」。
  - `ZSet`：要 **排序 + 按區間取**（`ZRANGE` / `ZREVRANGE` / `ZRANGEBYSCORE`）就選它。
- 常用命令記：`ZADD`、`ZINCRBY`、`ZREVRANGE key 0 9 WITHSCORES`、`ZREMRANGEBYRANK` / `ZREMRANGEBYSCORE` 做裁剪。

## 常見追問 / 陷阱

「排行榜同分怎麼辦？大 key 怎麼拆？」

→ score 可做成 `分數 + 時間小數` 打破平手；超大排行榜按業務分片（幣對/賽季/分區），避免單一 ZSet 過大拖慢。
