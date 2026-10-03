# Kafka：Consumer Rebalance

## 面試問題

Kafka Consumer Group 什麼時候會 rebalance？Eager vs Cooperative 差在哪？對 lag／重複消費有什麼影響？怎麼少觸發？

## 答法

**何時觸發**

- 成員**加入／離開**（啟動、宕機、滾動發布）。
- 超過 `session.timeout.ms`／`max.poll.interval.ms` 被協調器踢出。
- 訂閱的 topic **分區數變化**等。
- 目標：把 topic-partition **重新分配**給組內消費者。

**過程直覺**

- 停／撤銷舊分配（視協議）→ 算新分配 → 再消費。
- 窗口內可能短暫「某些分區沒人消費」→ **lag 短衝**。

**Eager vs Cooperative sticky**

| | Eager（舊 stop-the-world） | Cooperative sticky |
|---|---|---|
| 撤銷 | 先撤銷**全部**分區再重派 | 只撤銷「必須搬走」的分區 |
| 體感 | 全組停一下，抖動大 | 其餘分區可繼續，抖動小 |
| 方向 | 舊行為 | 新版 client 預設方向 |

**對 lag／重複的影響**

- 窗口內吞吐掉零／變慢 → lag 上升。
- 已 **commit** 的 offset 不丟；**未提交**的在新 owner 上可能**再消費一遍**（at-least-once 常態）。
- 坑多半是**重複、亂序窗口、本地狀態失效**，不是 broker 刪消息。

**怎麼減少不必要的 rebalance**

- 業務別堵住 poll：合理 `max.poll.interval.ms`／`max.poll.records`；處理加快或異步化。
- **靜態成員** `group.instance.id`：短時間重啟協調器會等，不立刻當新人踢走 → 滾動發布少抖。
- 用 **cooperative sticky**；狀態外置（Redis／DB），別假設「分區永遠黏同一實例」。
- 回調 `onPartitionsRevoked` 裡**別做重活／阻塞**。

**面試一句話**：成員進退／超時觸發重分配；Eager 全撤、Cooperative 只撤必要；未提交 offset 可能重複，用靜態成員＋cooperative＋別堵 poll 減震盪。

## 常見追問 / 陷阱

「rebalance 會不會丟消息？」

→ 提交過的不丟；未提交的可能**重複**。另一常見：滾動發布没用靜態成員／cooperative → 每次上下線全組抖、lag 突然飆。

## 小練習

三副本消費者滾動發布，每次重啟 lag 衝一波再回落。你會先開 `group.instance.id`、改 cooperative，還是先加大 `max.poll.interval.ms`？各解哪類問題？
