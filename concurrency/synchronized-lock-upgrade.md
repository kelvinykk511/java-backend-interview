# synchronized 底層 + 鎖升級（JDK 15+ 變化）

> 相關：[synchronized vs ReentrantLock](./synchronized-vs-reentrantlock.md)、[對象內存布局](../jvm/object-layout-compressed-oops.md)、[Safepoint](../jvm/safepoint.md)

## 面試問題

`synchronized` 底層點實現？講下鎖升級過程，同埋 JDK 15 之後有咩唔同？

## 模範答案

- **字節碼**：同步塊 = `monitorenter` + 兩個 `monitorexit`（第二個喺異常路徑，所以拋異常都會解鎖）；同步方法 = 方法上嘅 `ACC_SYNCHRONIZED` 標記。
- **鎖狀態記喺對象頭 Mark Word**。
- **升級路線（JDK 8 經典版）**：
  1. **無鎖**
  2. **偏向鎖**：Mark Word 記低第一個線程 ID，同一線程再入連 CAS 都唔使。
  3. 第二個線程嚟搶 → **撤銷偏向**（要等 safepoint，貴）→ **輕量級鎖**：線程喺自己棧幀建 Lock Record，CAS 將 Mark Word 換成指向佢；CAS 失敗就**自適應自旋**。
  4. 自旋都攞唔到 / 競爭激烈 → **膨脹重量級鎖**：`ObjectMonitor`（`_owner`、`_cxq`/`_EntryList` 排隊、`_WaitSet`），攞唔到就 park，涉及 OS 線程阻塞/喚醒。
- **基本唔降級**（HotSpot 只會喺後台/safepoint 回收空閒 monitor，面試講「升級為主、唔會倒返去偏向」即可）。
- **JDK 15（JEP 374）預設關閉並棄用偏向鎖，JDK 18 起相關參數失效**。原因：撤銷要 safepoint，成本高；現代代碼多用 JUC 容器，偏向收益細。所以而家實際路線係 **無鎖 → 輕量級 → 重量級**。
- **JIT 優化**：鎖消除（逃逸分析證明對象唔逃逸，例如方法內 `StringBuffer`）、鎖粗化（循環內反覆加解鎖合併）。

## 常見追問／陷阱

- **identity hashCode 同偏向鎖衝突**：Mark Word 冇位同時放 hash 同線程 ID → 計過 `System.identityHashCode`/未覆寫嘅 `hashCode()` 嘅對象唔會再偏向；已偏向時再計 hash 會撤銷甚至膨脹。
- **`wait/notify` 必須持有 monitor**，而且一用就會膨脹成重量級（要 `_WaitSet`）。冇持鎖調用拋 `IllegalMonitorStateException`。
- 「重量級一定慢」唔啱：臨界區長時，自旋只係燒 CPU，直接 park 反而好。
- **虛擬線程**：JDK 21 喺 `synchronized` 入面阻塞會 pin 住 carrier 線程；JDK 24（JEP 491）先解決。JDK 21 上高並發 IO 喺鎖內做，改用 `ReentrantLock`。
- 鎖對象唔好用 `Integer`/`String` 字面量（緩存/常量池共享，可能同其他代碼搶同一把鎖）。
