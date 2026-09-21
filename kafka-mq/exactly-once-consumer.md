# Kafka：消費端 Exactly-Once（對下游）

## 面試問題

開了冪等 Producer / Kafka 事務之後，消費者寫 MySQL 還會重複嗎？消費端怎麼做到「業務上」exactly-once？

## 答法

**先分清兩層**

- Kafka 內部 EOS：事務 Producer + `isolation.level=read_committed` → 讀到的是已提交、不重複的 log 視圖。
- **跨系統 EOS**：訊息進你的 DB / 下游仍是 at-least-once（重試、rebalance、手動重放）。

**消費端常見套路**

1. **關掉盲目 auto-commit**：先處理成功，再 `commitSync` offset（或用事務性消費把 offset 寫進 Kafka 事務）。處理失敗不提交 → 會再投遞，所以下游必須冪等。
2. **業務冪等鍵**：訂單號 / `txHash` / `eventId` 做 DB 唯一約束；撞鍵當「已成功」。
3. **Outbox / 本地訊息表**：與業務同一 DB 事務寫入「待發事件」，另進程可靠投遞（避免「寫庫成功但 offset 沒提交」或相反）。
4. **Consume-transform-produce**：讀 A、寫 B 都放進 Kafka 事務（`sendOffsetsToTransaction`）——只保證 Kafka↔Kafka；寫外部 DB 仍要 2 或 outbox。

**面試一句話**：Kafka EOS ≠ 你的帳務 EOS；消費端 = **處理成功才提交 offset + 下游冪等（唯一鍵/狀態機）**。

## 常見追問 / 陷阱

「enable.auto.commit=true 配手動業務冪等夠不夠？」

→ 常不夠穩：可能「業務已成功、offset 尚未提交」就 rebalance → 重複消費（冪等能扛）；也可能「offset 先提交、業務失敗」→ **丟訊息**（更糟）。生產環境傾向手動提交，且 **先副作用成功再 commit**。
