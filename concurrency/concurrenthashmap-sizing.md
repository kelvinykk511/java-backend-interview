# ConcurrentHashMap：容量、負載與擴容

## 面試問題

`ConcurrentHashMap` 的初始容量、負載因子怎麼理解？什麼時候觸發 resize？多執行緒同時 put 時擴容大致怎麼協作？

## 答法

**容量口訣（JDK8+）**

- 構造參數 `initialCapacity` 是**期望元素個數的量級**，不是「桶陣列長度就等於這個數」。內部會算成 **≥ 容量需求的 2 的冪** table size。
- **負載因子**預設 `0.75`：當元素數逼近 `capacity × loadFactor` 時準備擴容（實際用 `sizeCtl` 等閾值協調，面試講「約 0.75 觸發」即可）。
- 預估會塞很多 key 時，**給夠 initialCapacity**，避免反覆翻倍擴容；別一開始就塞超大空表（浪費記憶體）。

**為什麼要 2 的冪**

- 下標用 `hash & (n-1)`，比 `%` 快；擴容時元素要嘛留在原下標，要嘛 `+ oldCap`（高位 bit 決定），方便並行搬遷。

**多執行緒擴容（精簡版）**

1. 某個 put 發現該擴了 → 建立 **2 倍**新 table，用 `sizeCtl` 標記「正在轉移」。
2. 其他寫執行緒進來可以**幫忙搬**一段桶（transfer index 往下切），不是只有一條執行緒傻搬整張表。
3. 桶上可能掛 **樹化**（長鏈衝突、`treeify`）或 **ForwardingNode**（已搬完指向新表）——讀寫撞到會協助或转到新表。
4. 讀多寫少場景：多數時候無鎖讀；寫只鎖**單個桶頭**（synchronized 桶首節點），不是整表一把大鎖（別再背 JDK7 Segment 當唯一答案）。

**和 HashMap 比**：單執行緒 `HashMap` 擴容簡單；`ConcurrentHashMap` 要在不鎖全表的前提下完成搬遷，所以有協助轉移與轉發節點。

## 常見追問／陷阱

「`new ConcurrentHashMap(1000)` 就是 1000 個桶？」

→ **不是**。那是容量提示；實際 table 長度是算過後的 **2 的冪**，還要留 loadFactor 餘地。另一陷阱：用 CHM 當計數器只 `get` 再 `put` → 仍有競態；計數用 `AtomicInteger`／`LongAdder`，或 `merge`／`compute`。

「size() 絕對準？」→ 高併發下是**估算／弱一致**視角（實現會盡力統計）；強依賴精確計數別只靠頻繁 `size()`。

## 關聯

- [volatile 與 happens-before](volatile-happens-before.md)
- [synchronized vs ReentrantLock](synchronized-vs-reentrantlock.md)
