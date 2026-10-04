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
