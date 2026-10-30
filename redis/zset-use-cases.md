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

---

## 進階（2026-10-30）：底層編碼 + 用 ZSet 做延遲隊列

### 面試問題

訂單 15 分鐘未付款自動取消，用 Redis ZSet 做延遲隊列：score 放到期時間戳。多個實例一齊輪詢，點樣唔重複處理、唔丟任務？ZSet 底層係咩結構？

### 答法

- **底層編碼**：
  - 元素少（預設 `zset-max-listpack-entries 128`、單個值 ≤ `zset-max-listpack-value 64` 字節）→ **listpack**（Redis 7 前叫 ziplist），連續內存、慳位但插入 O(n)。
  - 超過閾值 → **skiplist + dict**：dict 做 member→score O(1)，skiplist 按 score 排序，查/插 O(log n)，範圍查詢沿底層鏈表走。
  - 點解用跳表唔用紅黑樹：範圍查詢簡單（找到起點順住走）、實現簡單、可以靠調層數概率平衡內存。
- **延遲隊列基本做法**：`ZADD delay_queue <到期ms> <orderId>`；worker 定時 `ZRANGEBYSCORE delay_queue 0 <now> LIMIT 0 100` 攞到期任務。
- **多實例防重複**：「查 + 刪」要原子，用 **`ZREM` 返回值搶佔**（返回 1 先係你嘅）或者 **Lua 腳本**一次過 `ZRANGEBYSCORE` + `ZREM` 搶一批。
- **防丟任務**（ZREM 之後 worker 掛咗）：
  - 搶到後唔係直接刪，而係搬去「處理中」ZSet（score = now + 超時），處理完再刪；有掃描器把超時嘅搬返入隊 → at-least-once。
  - 所以**業務必須冪等**：`UPDATE orders SET status='CANCELLED' WHERE id=? AND status='UNPAID'`，影響行數 0 就當已處理。
- **侷限**：輪詢有延遲 + 空轉；Redis 異步複製，主從切換可能丟最近寫入；無 ACK/重試/死信。量大或要求可靠 → 用 MQ 延遲消息（RocketMQ 延時 / 定時消息、Kafka 自建分級延遲 topic），DB 狀態做最終兜底掃描。

### 常見追問 / 陷阱

- **熱 key**：一個大 ZSet 全部實例打 → 按 `orderId` hash 分成 N 個 ZSet，每個 worker 負責幾片。
- **score 精度**：score 係 double，毫秒時間戳（~1.7e12）OK；但用「分數 × 1e13 + 時間」做排行榜同分打破平手會超過 2^53 精度丟失。
- `ZRANGEBYSCORE` 喺 6.2+ 可以用 `ZRANGE key min max BYSCORE LIMIT ...` 取代。
