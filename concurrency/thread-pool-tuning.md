# 線程池參數調優

**問題：** ThreadPoolExecutor 參數點設？線上線程池打滿點處理？

## 答案要點
- 提交順序：core 未滿開新線程 → 入隊列 → 隊列滿再開到 max → 再滿走拒絕策略。
- CPU 密集：core ≈ CPU 核數+1；IO 密集：≈ 核數 × (1 + 等待時間/計算時間)，最終靠壓測。
- 隊列一定要有界（`LinkedBlockingQueue` 預設無界 → OOM；`Executors` 工廠方法唔好用）。
- 拒絕策略：Abort（報錯）、CallerRuns（背壓）、Discard 類（會丟任務，要記錄）。
- 線程命名、監控 activeCount/queueSize/rejectCount；可以用動態線程池（配置中心改 core/max）。

## 追問／陷阱
- 隊列無界時 max 永遠唔生效。
- 父子任務共用同一個池 → 互相等待死鎖。
