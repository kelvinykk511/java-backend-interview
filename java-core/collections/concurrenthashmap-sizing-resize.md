# ConcurrentHashMap：size 與擴容

## 面試題

`ConcurrentHashMap` 的 `size()` 準不準？擴容（resize）怎麼保證併發安全？和 `HashMap` 有什麼差別？

## 模範答案（面試官向）

- **size()**：JDK 8+ 不是簡單的全局計數器。寫路徑用 `baseCount` + `CounterCell[]`（類似 LongAdder）累加；讀 `size()` 會 sum 這些計數。高併發下可能短暫不精確，但通常夠用；若要「瞬間精確」本身就跟鎖無關的併發結構衝突。
- **擴容觸發**：負載因子（預設 0.75）× 容量；或單桶樹化／鏈過長等條件也可能觸發。
- **併發擴容**：多執行緒可幫忙搬遷（transfer）。用 `forwarding` 節點標記已搬完的桶；讀到 forwarding 會協助或轉到新表。`sizeCtl` 用 CAS 協調「誰發起擴容／幾個 helper」。
- **vs HashMap**：`HashMap` 單執行緒擴容，併發寫會丟資料或死循環（JDK 7）；CHM 設計目標就是分段／桶級併發 + 可協助的 resize。
- **實務**：預估容量用構造參數 `initialCapacity`，減少 resize；別用 `size() == 0` 當嚴格同步條件。

## 常見追問／陷阱

「`mappingCount()` 和 `size()` 差在哪？」→ `mappingCount()` 回傳 `long`，大 map 時更合適；語意仍是估計值那一類。面試官也可能問：為什麼不用一個 `AtomicLong` 當 size？→ 熱點更新會變成單點瓶頸，所以拆成 cells。
