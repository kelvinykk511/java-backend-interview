# Java 對象內存布局 & 壓縮指針

## 面試題
`new Object()` 佔幾多 byte？點解 heap 由 31G 加到 40G，可用內存反而可能變少？

## 答案要點
- **對象 = 對象頭 + 實例數據 + 對齊填充**（整體對齊到 8 byte 倍數）。
  - Mark Word 8B：hashCode、GC 年齡（4 bit，所以 `MaxTenuringThreshold` 最大 15）、鎖狀態。
  - Klass pointer：開壓縮 4B，唔開 8B。
  - 數組多 4B 長度。
- **`new Object()` = 16B**：開壓縮時頭 12B → 補到 16B；唔開壓縮頭本身 16B。
- **壓縮指針（CompressedOops）**：引用用 32 bit 存「第幾個 8B 格」，所以最多尋址 2^32 × 8B = 32GB。
- **32G 陷阱**：heap 大過約 32G，JVM 自動關閉壓縮指針，每個引用由 4B 變 8B，對象全部變肥。31G 有壓縮可能裝得落多過 40G 冇壓縮。要大過 32G 就要加到明顯更大（例如 48G+）先划算，或者用 `-XX:ObjectAlignmentInBytes=16` 換到 64G 尋址但多填充。
- **字段重排**：JVM 會將 long/double 排前、引用排後，減少填充，所以源碼次序唔等於內存次序。
- **包裝類貴**：`int` 4B，`Integer` 16B + 引用 4B；`HashMap<Long, Long>` 每個 entry 連 Node + 兩個 Long 大概 60B+，海量數據用原生類型集合或數組。
- **點量**：用 JOL `ClassLayout.parseInstance(obj).toPrintable()`；確認壓縮：`java -XX:+PrintFlagsFinal -Xmx40g -version | grep UseCompressedOops`。

## 追問 / 陷阱
- 調用過 `System.identityHashCode()` / 冇覆寫嘅 `hashCode()` 之後，hash 會寫入 Mark Word。
- 新版本：JDK 24 起有實驗性 Compact Object Headers（JEP 450，`-XX:+UseCompactObjectHeaders`），對象頭壓到 8B，小對象多嘅服務可以慳 10%+ 內存。

## 一句話
對象頭 12B（壓縮）+ 8B 對齊；heap 過 32G 壓縮指針失效，引用翻倍。
