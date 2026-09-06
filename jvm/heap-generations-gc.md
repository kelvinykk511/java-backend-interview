# 堆分代與 Minor / Full GC

## 面試問題

JVM 堆為什麼分 Young / Old？Minor GC 和 Full GC 差在哪？

## 答法

- **分代假設**：大多數對象活很短；少數活很久。短命的放 Young，老對象晉升到 Old，掃起來更划算。
- **Young**：Eden + Survivor（S0/S1）。新對象先進 Eden；Minor GC 清 Young，倖存者在 S0/S1 之間來回，年齡夠了進 Old。
- **Old**：長壽對象；空間不夠或顯式觸發時做 Full GC（通常更慢、停頓更明顯）。
- **面試一句話**：Minor GC 快、頻；Full GC 慢、少——調優常盯 Full GC 次數與停頓。

## 常見追問 / 陷阱

「Minor GC 一定不會動 Old 區？」

→ 一般 Minor 主要掃 Young；但寫屏障 / 記憶集要處理「Old 引用 Young」。別說成「完全不管 Old」。

## 小練習

對象什麼時候會從 Young 晉升到 Old？說出兩個常見條件就行。
