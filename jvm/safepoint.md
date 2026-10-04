# JVM Safepoint（安全點）

## 面試問題

什麼是 Safepoint？為什麼「不是只有 GC」才會停頓執行緒？Time-to-Safepoint 跟 GC pause 差在哪？

## 答法

- **Safepoint**：JVM 要求「所有（或指定）Java 執行緒都停在可安全操作的點」——此時堆／執行緒狀態穩定，才能做全局操作。
- **常見觸發**：Stop-The-World GC、deoptimization、biased lock 撤銷、部分 JVMTI／heap dump、線程 dump（`jstack`）等。
- **執行緒怎麼停**：跑編譯碼時在「安全點輪詢」處檢查；跑解釋碼／阻塞在 JNI／鎖上時另有進入路徑。卡住進不了 safepoint 的執行緒會拖長 **Time-to-Safepoint（TTSP）**。
- **TTSP ≠ GC pause**：應用「停住」的總時間 ≈ **到達 safepoint 的等待** + **safepoint 內實際工作**（含 GC）。Young GC 本身 5ms，但若某執行緒卡在長 JNI／少輪詢熱迴圈，TTSP 可能先燒掉幾十 ms——日誌裡看起來像「莫名 STW」。
- **日誌怎麼分**：舊旗標 `-XX:+PrintGCApplicationStoppedTime`（Stopped 含進點＋點內）；統一日誌看 `safepoint`／`gc` 相關 tag，對比「到達時間」與「GC 實際耗時」。別只看 GC pause 數字就斷定全是回收器。
- **面試一句話**：Safepoint 是 JVM 做全局動作前的「全員集合點」；線上偶發長停頓不一定是 GC，先分清 TTSP 跟點內工作。

## 常見追問／陷阱

「把 GC 停頓調小就不會卡了？」／「`jstack` 本身安全嗎？」

→ 不一定。長 JNI、數值熱迴圈少輪詢會拉長 TTSP，看起來像莫名 STW。另：**`jstack`／線程 dump 本身也要進 safepoint**——業務尖峰時「dump 卡很久才出結果」常是 TTSP 慢的症狀，不是 dump 工具壞了；此時 dump 還會再疊一層短暫 STW，排查要避開尖峰或先看 safepoint／StoppedTime 日誌。

## 小練習

線上 `jstack` 偶發卡很久才出結果，同時業務延遲尖峰；你會先看哪些指標／日誌區分「GC 停頓」和「Time-to-Safepoint 慢」？
