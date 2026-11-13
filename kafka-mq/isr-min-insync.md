# Kafka ISR 與 min.insync.replicas

## 面試問題

`acks=all` 跟 ISR、`min.insync.replicas` 是什麼關係？Leader 掛了為什麼還可能丟消息？

## 答法

- **ISR（In-Sync Replicas）**：跟得上 Leader 的副本集合（含 Leader）。落後太多會被踢出 ISR。
- **`acks=all`**：要等「**當前 ISR** 裡所有副本」都確認——**不是**等「`replication.factor` 配置的全部副本數」。ISR=2 時 all 只等這 2 個。
- **`min.insync.replicas`**：ISR 人數少於此值時，`acks=all` 的寫入會失敗（寧可拒寫，也不在過瘦的 ISR 上假裝成功）。
- **可靠性組合**：資金／訂單類常見 `acks=all` + `min.insync.replicas=2`（三副本裡至少兩份同步）+ 冪等 Producer；消費端仍按至少一次做業務冪等。
- **`unclean.leader.election`**：若允許非 ISR 選成新 Leader，可能丟已 ack 的數據。生產環境通常關掉；關 unclean 後，ISR 過瘦時可能**選不出 Leader → 不可寫／不可讀**，用可用性換一致性。

## 常見追問／陷阱

「三副本 + `acks=all` 就絕對不丟？」／「ISR 縮了 producer 為什麼報錯？」

→ 若 ISR 縮到 1，而 `min.insync.replicas=1`，Leader 一掛、新 Leader 又選了落後副本，窗口內仍可能丟。`acks=all` 必須和 ISR 人數、`min.insync.replicas`、以及是否允許 unclean election 一起講。

**實戰調試角**：監控見某 partition ISR 從 3→1，同時 producer 報 `NotEnoughReplicas`／`NotEnoughReplicasAfterAppend`——通常是 **`min.insync.replicas` 擋寫**（ISR 已低於門檻），不是業務碼突然壞了。先查 replica 落後原因（磁碟／網路／broker GC），再決定是否暫時降 `min.insync`（有丟數風險）或修副本。

## 小練習

監控發現某 partition 的 ISR 從 3 掉到 1，同時 producer 開始報錯。可能是哪兩個配置／現象在擋寫？你會先查 broker 還是網路？`unclean.leader.election=false` 時這段窗口的可用性表現是什麼？


---

## 深入（2026-11-13）：acks=all 等嘅係 ISR，唔係副本數

### 面試可以點追

「成交／出金 topic 開咗三副本同 `acks=all`，係咪就唔會丟消息？ISR 縮水時 producer 報 `NotEnoughReplicas` 應該點判斷？可唔可以臨時把 `min.insync.replicas` 改 1、打開 unclean election 頂住寫入？」

### 答法

**三件套要一齊講**

- **ISR**：跟得上 Leader 嘅副本集合（含 Leader）。落後超過 `replica.lag.time.max.ms` 等閾值會被踢出；追上回 ISR。
- **`acks=all`（-1）**：等**當前 ISR 全員**確認，**唔係**等 `replication.factor` 配置嘅全部副本。ISR=2 時，all 只等呢 2 個。
- **`min.insync.replicas`**：ISR 人數低於此值，`acks=all` 寫入失敗（`NotEnoughReplicas`／`NotEnoughReplicasAfterAppend`）。寧可拒寫，也不在過瘦 ISR 上假裝成功。

資金／成交／出金常見組合：`RF=3` + `acks=all` + `min.insync.replicas=2` + 冪等 producer；消費端仍按至少一次做業務冪等。`acks=all` 本身**唔等於** Exactly-Once，更唔等於「三份都落咗盤」。

**Leader 掛咗點解仲可能丟**

- 若 ISR 已縮到只剩 Leader，而 `min.insync.replicas=1`，`acks=all` 實際上只等 Leader 寫本地 log——耐用性接近 `acks=1`。Leader 一掛、未複製嘅已 ack 消息就會冇。
- **`unclean.leader.election=true`**：允許非 ISR（落後副本）選成新 Leader → 用舊 log 覆蓋，**已 ack 都可能丟**。生產通常關掉。
- 關 unclean 之後：ISR 過瘦或冇合格副本時可能**選不出 Leader** → 暫時不可寫／不可讀。呢個係用可用性換「唔用落後副本食掉已確認數據」。

**ISR 3→1 同時 producer 報錯**

- 多數係 `min.insync.replicas` 擋寫，唔係業務碼突然壞。
- 先查 replica 落後原因：磁碟、網絡、broker GC／長停頓、是否有 broker 掉線。修副本、等佢回 ISR。
- 臨時把 `min.insync.replicas` 降到 1，或打開 unclean election，等於主動接受丟數窗口。成交／出金 topic **唔好**用呢招頂流量；行情類可丟數據先另說。

### 追問／陷阱

「三副本 + `acks=all` 絕對不丟？」→ 要看當時 ISR 人數、`min.insync.replicas`、同是否允許 unclean。另一個：冪等 producer 只防重試導致嘅重複序號，**救唔到** unclean 選主造成嘅數據倒退。監控要睇 ISR 大小、under-replicated partitions，唔好只睇 producer 成功率。
