# Kafka：Consumer Rebalance

## 面試題

Kafka Consumer Group 什麼時候會 rebalance？過程中常見坑是什麼？怎麼降低影響？

## 模範答案（面試官向）

- **何時觸發**：成員加入／離開（啟動、宕機、`max.poll.interval.ms` 超時被踢）、訂閱的 topic 分區數變化、協調器認為成員失效等。目標是把 topic-partition **重新分配**給 group 內消費者。
- **過程直覺**：停消費 → 交回／撤銷舊分配（視協議與回調）→ 算新分配 → 再開始。期間可能出現短暫「誰都不消費某些分區」。
- **常見坑**：
  - 業務處理太慢 → poll 間隔過長 → 被踢 → 反覆 rebalance（抖動）。
  - 在 rebalance 回調裡做重活／阻塞。
  - 依賴「分區永遠黏在同一個實例」做本地狀態卻沒有處理 revoke。
- **緩解**：合理設 `max.poll.interval.ms` / `max.poll.records`；處理加快或異步化；靜態成員（`group.instance.id`）減少短暫閃斷重平衡；合作式協議（cooperative sticky）減少「全部撤銷再重派」的震盪；狀態外置（Redis/DB）別只靠進程內。

## 常見追問／陷阱

「rebalance 會不會丟消息？」→ 提交過的 offset 不丟；未提交的可能重複消費（at-least-once 常態）。坑多半是重複、亂序窗口、或本地緩存失效，不是 broker「刪掉消息」。
