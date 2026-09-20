# Redis Stream：XREADGROUP 與 PEL（面試加深）

## 面試問題

Consumer Group 裡 `XREADGROUP` 讀到的訊息，為什麼崩潰重啟後可能再讀到？PEL 是什麼？

## 答法

- **XREADGROUP GROUP g c STREAMS key >**：從 group 視角讀「還沒分給任何人」的新訊息（`>`），並記到該 consumer 名下。
- **PEL（Pending Entries List）**：已讀出但尚未 `XACK` 的訊息清單。崩潰、超時、沒 ACK → 訊息仍在 PEL。
- **至少一次**：業務必須可冪等；靠 `XACK` 宣告處理完成。
- 故障接手：`XPENDING` 看卡住的，再用 `XCLAIM`（或自動 claim）把別的 consumer 的 pending 搶過來重試。
- 跟 List 對比一句：List 的 `BRPOP` 取出即消失；Stream + group 是「讀出 → 處理 → ACK」三步，中間可恢復。

## 常見追問 / 陷阱

「讀了就當消費成功？」

→ 錯。沒 `XACK` 就只是進了 PEL。另一陷阱：一直用 `>` 只拿新訊息，**忽略 PEL** → 崩潰期間的訊息永遠沒人重試。正確姿勢：啟動時先處理 pending，再讀新訊息；或定時 claim 超時 pending。
