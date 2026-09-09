# ThreadPoolExecutor 核心參數

## 面試問題

講一下 `ThreadPoolExecutor` 的核心參數，任務進來後執行順序是什麼？

## 答法

常用構造參數：

| 參數 | 意思 |
|------|------|
| `corePoolSize` | 常駐核心執行緒數 |
| `maximumPoolSize` | 允許的最大執行緒數 |
| `keepAliveTime` | 非核心執行緒空閒多久回收 |
| `workQueue` | 任務排隊佇列 |
| `threadFactory` | 怎麼建執行緒（好用：統一命名） |
| `handler` | 佇列滿且執行緒到頂時的拒絕策略 |

**執行順序（常考）**

1. 執行緒數 < core → 直接開新執行緒跑
2. 已達 core → 任務進佇列
3. 佇列滿 → 再開執行緒，直到 max
4. 還進不來 → 走拒絕策略（Abort / CallerRuns / Discard / DiscardOldest）

實務：IO 密集可多一點執行緒；CPU 密集約 `CPU 核數 ± 1`。佇列別無限大，否則永遠開不到 max，也容易 OOM。

## 常見追問 / 陷阱

「佇列用 `LinkedBlockingQueue` 不設容量會怎樣？」

→ 幾乎永遠在步驟 2，max 形同虛設；負載高時堆積、延遲暴漲。生產環境要有界佇列 + 明確拒絕/降級。
