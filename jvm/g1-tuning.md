# G1 調優與面試答法

> 前置：[G1 GC 基礎](g1-gc-basics.md)

## 面試問題

G1 服務 P99 尖刺超過 300ms，GC 日誌出現 `to-space exhausted` 同 Full GC，你點調？

## 答法

1. **先開日誌分類問題**：`-Xlog:gc*`（或 JDK 8 用 `-XX:+PrintGCDetails`），睇係邊一類：
   - Young / Mixed GC 停頓本身太長
   - **Evacuation failure**（`to-space exhausted`）
   - **Humongous** 對象塞爆 Old
   - **Full GC**（通常係前面問題嘅結果）

2. **`MaxGCPauseMillis`（預設 200ms）**  
   - 係**目標**，唔係保證；設得太低 → Young 縮細 → GC 更密 → 吞吐跌、更容易 evacuation failure。  
   - **唔好**手動 `-Xmn` / `NewRatio` 鎖死 Young 大小，會打爛 G1 按停頓目標自動調 Region 嘅邏輯。

3. **`to-space exhausted`（疏散失敗）**  
   並發標記開得太遲、保留空間唔夠 evacuate：  
   - 調低 `InitiatingHeapOccupancyPercent`（JDK 9+ 多數自適應，仍可微調）  
   - 提高 `G1ReservePercent`（預設 10%）  
   - 適當加 `ConcGCThreads` 加快並發標記

4. **Humongous**  
   大過半個 Region 嘅對象直接進 Old、唔走 Young：  
   - 可試 `-XX:G1HeapRegionSize`（1–32MB，2 嘅次方）  
   - **更好**係改代碼：巨型 `byte[]`、一次 serialize 成個 JSON 等

5. **堆同容器**  
   - `Xms = Xmx`，避免運行中擴堆帶來額外成本  
   - 容器用 `-XX:MaxRAMPercentage`，唔好硬抄實體機 `-Xmx`

6. **停頓仍然唔夠**  
   評估 [ZGC vs G1](zgc-vs-g1.md)（JDK 21+ 有分代 ZGC，大堆尾延遲往往更穩）。

## 常見追問 / 陷阱

「G1 Full GC 仲係好慢？」

→ JDK 10 起 G1 Full GC 已係並行，但**仍然 STW 掃成個堆**，代表容量或配置出問題（evacuation 失敗、Humongous、堆太小），唔係靠再壓 `MaxGCPauseMillis` 就解決。

陷阱：一見 P99 高就 `MaxGCPauseMillis=50` —— 通常令 GC 更密、問題惡化，要先靠日誌分類再動參數。

## 小練習

Mixed GC 停頓 OK，但每週幾次 `to-space exhausted` → Full GC。你會先改邊三個參數／代碼方向？點樣驗證改完有效？
