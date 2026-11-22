# ConcurrentHashMap put 流程（JDK8+）

## 面試問題

`ConcurrentHashMap.put` 一個 key，內部大致經過咩步驟？點樣做到唔鎖全表都線程安全？

## 答法

1. **key/value 唔可以係 null**（直接 NPE）。原因：併發下 `get` 返回 null 分唔清「冇呢個 key」定「value 係 null」。
2. **算 hash**：`spread(h) = (h ^ (h >>> 16)) & HASH_BITS`，高位混入低位，最高位清 0（負 hash 留俾特殊節點用）。
3. **table 未初始化** → `initTable()`，用 CAS 改 `sizeCtl = -1` 搶初始化權，搶唔到就 `Thread.yield()` 等。
4. **桶係空** → 直接 **CAS** 放新節點，**唔加鎖**。CAS 失敗就重新循環。
5. **桶頭 hash == MOVED (-1)**（ForwardingNode）→ 表正擴容緊，`helpTransfer` 幫手搬，搬完再插去新表。
6. **桶有節點** → `synchronized(桶頭節點)`，只鎖呢一個桶：
   - 鏈表：行到尾，相同 key 就覆蓋，冇就尾插；順手數 `binCount`。
   - 紅黑樹（`TreeBin`）：走樹插入。
7. **鏈長 ≥ 8** → `treeifyBin`；但 **table < 64 時優先擴容**，唔樹化。
8. **`addCount`**：計數用 `baseCount` + `CounterCell[]`（類似 LongAdder，分散熱點），再檢查要唔要擴容。

一句總結：**空桶 CAS、有節點鎖桶頭、撞到擴容就幫手搬、計數分段累加**。

## 常見追問／陷阱

- 「JDK7 點做？」Segment 分段鎖（繼承 ReentrantLock），併發度固定 = segment 數；JDK8 改為桶級鎖 + CAS，粒度更細。JDK8 構造參數 `concurrencyLevel` 只係容量提示，唔再係分段數。
- 「`size()` 準唔準？」係**估算**：併發寫緊時只係某一刻的近似值。要準確計數就自己維護。
- 「`get` 要唔要鎖？」唔使。`Node.val`、`next` 係 `volatile`，table 讀用 `Unsafe.getObjectVolatile`。
- 陷阱：`if (!map.containsKey(k)) map.put(k, v)` 唔係原子操作，要用 `putIfAbsent` / `computeIfAbsent`。
- 陷阱：`computeIfAbsent` 入面再改同一個 map（尤其同一個桶）可能死循環/拋 `IllegalStateException`（JDK8 有 bug，JDK9 改為報錯）。
- 陷阱：用無界 CHM 做本地緩存／去重表，key 唔淘汰就會一直脹堆。進程內 map 亦**唔係**跨實例冪等；下單去重要落 DB 唯一鍵。

## 深入：sizeCtl、協助擴容、compute 陷阱（2026-11-22 refresh）

**面試可以再問**：多線程同時 put 觸發擴容時點協作？點解 `containsKey` + `put` 會重複下單？

- **`sizeCtl`**：`0` 未初始化；`-1` 有人正在 `initTable`；負值且絕對值 >1 時表示正在 transfer（入面編碼咗 stamp 同參與搬遷嘅 worker 數）；正值係下次擴容閾值（大約容量 × 0.75）。
- **協助搬遷**：某個寫線程建 2 倍新表之後，其他寫入撞到 ForwardingNode 或發現正在 resize，會按 `transferIndex` 分段（stride）搬桶。已搬嘅桶掛 ForwardingNode，讀寫都去新表。唔係單一線程搬整張表。
- **`containsKey` 再 `put`**：兩條線程可以同時見到「冇呢個 key」，然後各自 `put`——後 put 覆蓋先 put，但兩邊都以為自己係第一次，業務就跑咗兩次。要用 `putIfAbsent`／`computeIfAbsent` 一次過，而且只係**單進程**互斥。
- **`computeIfAbsent`**：mapping function 應係純計算；入面再 `put` 同一個 map，JDK 8 可能喺同一桶上重入 `synchronized` 或形成環，JDK 9+ 較多直接丟 `IllegalStateException`。相關 key 嘅寫要放喺 function 外面，或合併成一次 `compute`。
- CHM 適合進程內高併發字典，唔適合當分佈式鎖或跨實例去重。

## 相關

- [ConcurrentHashMap 容量與擴容](concurrenthashmap-sizing.md)
- [HashMap 擴容/樹化](../java-core/hashmap-resize-treeify.md)
