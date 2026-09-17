# G1 GC 基礎與面試答法

## 面試問題

G1 跟舊的 Parallel / CMS 差在哪？面試時怎麼講「可預期停頓」？

## 答法

- **Region 模型**：堆切成很多等大 Region（Young / Old / Humongous），不是整塊 Young/Old 連續區。
- **目標**：盡量把停頓壓在你設的目標內（例如 `-XX:MaxGCPauseMillis`），用「先清垃圾最多的 Region」換時間。
- **流程粗分**：並發標記（找活對象）→ 混合回收（清部分 Old + Young）→ 必要時 Full GC（要避免）。
- **面試一句話**：G1 用 Region + 優先清「性價比高」的區，換可預期停頓；調優先盯停頓目標與 Humongous / Full GC，別一開始狂調一堆參數。

## 常見追問 / 陷阱

「把 MaxGCPauseMillis 設很小就一定更快？」

→ 停頓目標更嚴 → 可能更頻繁 GC、吞吐量下降。目標是「夠用的停頓」，不是越小越好。Humongous（超大對象占多 Region）也常是隱藏殺手。

## 小練習

線上偶發長停頓，日誌出現 `Full GC (Allocation Failure)` 與大量 Humongous。你會先查哪兩件事？
