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

---

## 深入（2026-11-16）：`>` vs PEL、consumer name、ACK 時機

### 面試可以點追

充值入帳用 Redis Stream Consumer Group：`XREADGROUP GROUP credit worker-1 STREAMS deposits >`，處理成功先 `XACK`。一個 Pod 被殺重啟後，部分充值一直未入帳，但 Stream 仲有新消息，`XPENDING` 見到一堆 pending。點解 `>` 讀唔返呢啲消息？應該點恢復？ACK 應該喺入帳前定後？

### 答法

**`>` 同 PEL 係兩條路**

- `>` 只取「呢個 group **從未派發過**」嘅新消息，派出同時記入該 consumer 嘅 **PEL（Pending Entries List）**。
- 已經派咗但未 `XACK` 嘅，只停喺 PEL，`>` **唔會再派**。所以崩潰重啟後若只繼續讀 `>`，舊消息會一直掛住，看起來似「丟咗」，其實仲喺 pending。
- 正確恢復：
  - consumer name **穩定**（例如 `worker-0`，唔好帶 pid／隨機數）：啟動時先 `XREADGROUP ... 0-0` 讀自己嘅 PEL，處理完 `XACK`，再轉返 `>`。
  - name 變咗（發版、擴縮容、hostname+pid）：舊 PEL 仍掛喺舊名字下面，新進程讀自己 PEL 係空嘅。要 `XPENDING` + `XCLAIM`／`XAUTOCLAIM` 按 idle 認領，再處理 + ACK。
- 語意係 **at-least-once**：處理成功（入帳落庫）**之後**先 ACK。崩潰喺 ACK 前會重投，入帳必須按 `depositId` **冪等**。先 ACK 再寫庫係 at-most-once，進程一掛就丟消息，**資金路徑唔可以**。

**同 List、同 Kafka 差在邊**

- List 嘅 `BRPOP` 取出即消失，冇 group、冇 PEL，進程掛咗就要靠自己另外做可靠隊列。
- Stream ≠ Kafka：單一 stream 係**一條 Redis key**，受單線程／單 slot 限制，高吞吐要自己按用戶或幣對切多條 stream。持久化靠 Redis 記憶體 + AOF／RDB，冇 Kafka 那套分區重平衡。

### 追問／陷阱

- 一直用 `>`、忽略 PEL：死掉嘅 consumer 手上消息永遠冇人重試；PEL 只會越積越長，佔記憶體，新消息讀取亦會受影響。
- `XCLAIM`／`XAUTOCLAIM` 嘅 min-idle **短過**最長處理時間：worker A 仲寫緊庫，B 已經把同一筆搶過去再入帳一次 → 一定要冪等，idle 要大於處理超時 + 緩衝。
- `XADD MAXLEN` 把未 ACK 嘅條目修掉，PEL 仲掛住 ID，claim 會拿到空消息。要定期清呢啲 stale pending（確認已經唔喺 stream 再 `XACK` 掉），資金路徑修剪要保守。
- `NOACK` 等於放棄 PEL，崩潰即丟，唔好用喺入帳。
