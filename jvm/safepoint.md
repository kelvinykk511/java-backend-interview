# JVM Safepoint（安全點）

## 面試問題

什麼是 Safepoint？為什麼「不是只有 GC」才會停頓執行緒？

## 答法

- **Safepoint**：JVM 要求「所有（或指定）Java 執行緒都停在可安全操作的點」——此時堆／執行緒狀態穩定，才能做全局操作。
- **常見觸發**：Stop-The-World GC、deoptimization、biased lock 撤銷、部分 JVMTI／heap dump、線程 dump（`jstack`）等。
- **執行緒怎麼停**：跑編譯碼時在「安全點輪詢」處檢查；跑解釋碼／阻塞在 JNI／鎖上時另有進入路徑。卡住進不了 safepoint 的執行緒會拖長「到點時間」（Time to safepoint）。
- **面試一句話**：Safepoint 是 JVM 做全局動作前的「全員集合點」；線上偶發長停頓不一定是 GC，也可能是進 safepoint 慢或 safepoint 操作本身重。

## 常見追問／陷阱

「把 GC 停頓調小就不會卡了？」

→ 不一定。若大量執行緒卡在無法快速進入 safepoint 的路徑（長 JNI、數值熱迴圈少輪詢），`Time to safepoint` 會變長，看起來像「莫名 STW」。要用 GC／safepoint 日誌分開看，不要只怪 Young GC。

## 小練習

線上 `jstack` 偶發卡很久才出結果，同時業務延遲尖峰；你會先看哪些指標區分「GC 停頓」和「進 safepoint 慢」？
