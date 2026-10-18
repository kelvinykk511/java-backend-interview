# Redis 主從複製（全量 / 增量 / PSYNC）

## 面試問題

Redis 主從複製點運作？從庫斷線重連後一定要全量同步嗎？

## 答法

### 首次同步（全量）

1. replica 發 `PSYNC ? -1`
2. master 回 `FULLRESYNC <replid> <offset>`
3. master `bgsave` fork 出 RDB 發給 replica；期間新寫命令暫存喺該 replica 的 **replication buffer**（client output buffer）
4. replica 清空舊數據、載入 RDB → 回放 buffer → 之後持續命令傳播（**異步**）

### 斷線重連（增量）

- replica 發 `PSYNC <replid> <offset>`
- **replid 對得上**且 **offset 仍喺 `repl_backlog`**（環形緩衝，預設 1MB）範圍內 → master 回 `CONTINUE`，只補缺口
- 否則 → 全量

### PSYNC2（4.0+）

- 切主後新主保留舊 **replid** 做 **replid2**；其他從庫可以增量接上
- 正常重啟可從 RDB 讀回 replid / offset

### 異步複製與 WAIT

- 主宕機時未傳到從庫的寫會丟
- `WAIT numreplicas timeout` 可等 N 個副本 ack，但仍**唔等於強一致**

### 調參（面試常問）

| 參數 | 要點 |
|------|------|
| `repl-backlog-size` | ≈ 寫入速率 × 可容忍斷線秒數 × 2（例：每秒 2MB、容忍 60 秒 → 至少 ~240MB） |
| `client-output-buffer-limit replica` | 太細 → 全量期間 buffer 爆 → 連接被斷 → 再全量，形成死循環 |
| `repl-diskless-sync` | 大實例可考慮無磁碟全量同步 |

### 從庫過期 key

- 從庫唔主動刪，等主庫 `DEL` 傳播
- 3.2+ 讀從庫時已過期 key 會返回不存在

**面試一句話**：首次全量 RDB + buffer；斷線靠 replid + offset 對 backlog 做增量；對唔上就全量；複製默認異步，要控丟窗靠 WAIT 或業務設計。

## 常見追問 / 陷阱

**(a) 讀寫分離**

→ 會讀到舊數據（複製延遲）。要求讀己之寫嘅場景：**讀主**或**帶版本號**。

**(b) 主從 ≠ 高可用**

→ Failover 要靠 **Sentinel** / **Cluster**；主從只解決讀擴展同備份。

**(c) 腦裂**

→ 被隔離嘅舊主繼續收寫，恢復後變從庫會被**全量覆蓋** → 寫入丟失。用 `min-replicas-to-write 1` + `min-replicas-max-lag 10` 令失去從庫嘅主**拒寫**，把丟失窗口限制喺約 10 秒內。

## 小練習

主從延遲 200ms，用戶改密碼後立刻用舊 token 打從庫校驗 —— 你會怎麼設計路由或版本校驗？
