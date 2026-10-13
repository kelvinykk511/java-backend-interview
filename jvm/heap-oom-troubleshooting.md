# 線上 OOM（Java heap space）排查

**Q:** 線上服務報 `java.lang.OutOfMemoryError: Java heap space` 之後掛咗，重啟就好，但隔幾日又再出。你點樣搵根因？

## 先分清係邊種 OOM（睇錯誤訊息後半句）
- `Java heap space`：堆滿，最常見。
- `GC overhead limit exceeded`：GC 用咗 >98% 時間但收唔返 <2% 內存，本質都係堆唔夠/洩漏。
- `Metaspace`：類太多（動態代理、Groovy、熱部署），見 [metaspace-oom](metaspace-oom.md)。
- `Direct buffer memory`：Netty/NIO 堆外內存，堆 dump 睇唔到，要查 `-XX:MaxDirectMemorySize`、ByteBuf 冇 release。
- `unable to create native thread`：線程開太多（線程池無上限、`ulimit -u`），唔係堆問題。

## 標準步驟
1. **預先配好參數**（事後先加就遲咗）：`-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/dump/`，加 GC 日誌。
2. 拎到 `.hprof` 用 MAT / VisualVM 打開：
   - **Leak Suspects** 報告先睇一眼。
   - **Dominator Tree** 按 Retained Heap 排序，搵佔最大的對象。
   - 對可疑對象睇 **Path to GC Roots**（排除弱/軟引用），睇係邊個一直拎住佢。
3. 未掛但懷疑緊：`jstat -gcutil <pid> 1000` 睇老年代 (O) 每次 Full GC 後係咪越嚟越高 → 洩漏；`jmap -histo:live <pid> | head` 睇邊類對象數量暴升。
4. 區分「洩漏」vs「真係唔夠」：Full GC 後老年代回落到穩定水位 = 唔夠/突發大對象；持續爬升唔落 = 洩漏。

## 常見元兇
- 靜態 `Map` 當緩存冇淘汰、冇上限（用 Caffeine 設 maximumSize/expire）。
- `ThreadLocal` 喺線程池入面用完冇 `remove()`。
- 一次過查大量數據：`SELECT *` 冇分頁、導出 Excel 全部讀入內存（改流式/分批）。
- 無界隊列堆積：`LinkedBlockingQueue` 預設無界，下游慢就無限堆任務。
- 監聽器/回調註冊咗冇註銷；內部類拎住外部大對象。

## 追問 / 陷阱
- 「點解唔直接加大 -Xmx？」→ 洩漏情況下加大只係延遲爆炸，仲令 Full GC 停頓更長。
- 「容器入面點設堆？」→ 用 `-XX:MaxRAMPercentage=75` 跟容器 limit，留位畀 Metaspace、線程棧、Direct memory，否則可能被 cgroup OOM-kill（呢個冇 Java 錯誤，只見 `exit 137`）。
- 大堆 `jmap -dump` 會 STW 好耐，要先將實例從負載均衡摘走。
