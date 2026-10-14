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

- 「JDK7 點做？」Segment 分段鎖（繼承 ReentrantLock），併發度固定 = segment 數；JDK8 改為桶級鎖 + CAS，粒度更細。
- 「`size()` 準唔準？」係**估算**：併發寫緊時只係某一刻的近似值。要準確計數就自己維護。
- 「`get` 要唔要鎖？」唔使。`Node.val`、`next` 係 `volatile`，table 讀用 `Unsafe.getObjectVolatile`。
- 陷阱：`if (!map.containsKey(k)) map.put(k, v)` 唔係原子操作，要用 `putIfAbsent` / `computeIfAbsent`。
- 陷阱：`computeIfAbsent` 入面再改同一個 map（尤其同一個桶）可能死循環/拋 `IllegalStateException`（JDK8 有 bug，JDK9 改為報錯）。

## 相關

- [ConcurrentHashMap 容量與擴容](concurrenthashmap-sizing.md)
- [HashMap 擴容/樹化](../java-core/hashmap-resize-treeify.md)
