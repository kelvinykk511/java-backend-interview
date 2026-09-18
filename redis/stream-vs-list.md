# Redis Stream vs List

## 面試問題

消息隊列場景下，Redis `List` 和 `Stream` 怎麼選？各缺什麼？

## 答法

- **List（`LPUSH`/`BRPOP`）**：簡單佇列。多消費者搶同一條（競爭消費），沒有消費組、沒有單條 ACK、沒有持久遊標。適合「丟進就忘、丟了可重做」的輕量任務。
- **Stream**：專為消息日誌設計。有 **Consumer Group**、每消費者獨立 pending、`XACK`、`XREADGROUP`、可按 ID 回溯。更接近「至少一次 + 可追蹤」的 MQ 用法。
- **對比記法**：
  | | List | Stream |
  |---|---|---|
  | 消費模型 | 搶奪式 | 組內分配 + pending |
  | ACK | 無 | `XACK` |
  | 回溯 / 歷史 | 彈出即沒 | 可按 ID 讀 |
  | 複雜度 | 極簡 | 稍重，功能完整 |
- CEX 後端：內部異步任務、限流緩衝常用 List；要「誰消費了、失敗重試、組內負載」就用 Stream，或直接上 Kafka。

## 常見追問 / 陷阱

「Stream 保證 exactly-once 嗎？PEL 是什麼？」

→ 不保證 exactly-once，是 **at-least-once**（未 ACK 會留在 Pending Entries List）。消費者要業務冪等。PEL 過長要查卡死消費者（`XPENDING` / `XCLAIM`）。
