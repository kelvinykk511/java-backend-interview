# Redis Stream vs List

## 面試問題

消息隊列場景下，Redis `List` 和 `Stream` 怎麼選？各缺什麼？Stream 算不算 Kafka？

## 答法

**List：極簡隊列**

- 典型：`LPUSH` + `RPOP`／`BRPOP`（或對側方向），多消費者**搶同一條**（競爭消費）。
- **沒有**消費者組、**沒有** pending 清單、ACK 要自己做（或靠可靠隊列套路另拼）。
- 適合：丟進就忘、丟了可重做、單消費者／極簡 fan-out、不需要按 ID 回溯。

**Stream：近似 MQ 語義**

- 寫入：`XADD`；消費：消費者組 + `XREADGROUP`。
- **PEL**（Pending Entries List）+ `XACK`：未 ACK 留在 pending → **at-least-once** 模型清楚；超時用 `XPENDING`／`XCLAIM` 認領。
- 可按消息 ID 回溯／重讀歷史；多**消費者組**各自進度。

**對比記法**

| | List | Stream |
|---|---|---|
| 消費模型 | 搶奪式 | 組內分配 + pending |
| ACK | 無（自己做） | `XACK` |
| 回溯／歷史 | 彈出即沒（除非另存） | 可按 ID 讀 |
| 複雜度 | 極簡 | 稍重，語意更完整 |

**何時用哪個**

- **List**：極簡 fan-out、單消費者、不需要回溯／pending 認領、失敗可整任務重丟。
- **Stream**：多消費者組、要 `XPENDING`／`XCLAIM`、要按 ID 回溯、要「至少一次 + 可追蹤」的近似 MQ。

**CEX 後端類比**

- 內部輕量任務（刷新快取、非關鍵通知緩衝）→ **List** 夠用。
- 要「至少一次 + 認領超時／卡死接管」→ **Stream**；跨服務、要分區重平衡／長期堆積／強審計 → 上 **Kafka**。

## 常見追問 / 陷阱

「Stream 保證 exactly-once 嗎？PEL 是什麼？」

→ 不保證 exactly-once，是 **at-least-once**（未 ACK 留在 PEL）。消費者要**業務冪等**。PEL 過長先查卡死／慢消費者（`XPENDING`／`XCLAIM`）。

「Stream = Kafka？」

→ **不是**。Stream 沒有 Kafka 那套分區重平衡、消費者組協議與叢集級持久模型；容量／持久靠 **Redis 本身**（記憶體與 AOF／RDB）。List + `BRPOPLPUSH`／`BLMOVE` 可做可靠隊列，但仍比 Stream 的組＋pending 模型弱。

## 小練習

內部「掃鏈上確認數」任務：單 worker 重試可接受、不需要組內認領——List 還是 Stream？若要多組獨立進度（風控一組、入帳一組）呢？
