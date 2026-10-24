# CopyOnWriteArrayList

## 面試題
讀多寫少嘅 list 點揀？CopyOnWriteArrayList 原理同陷阱？

## 答案要點
- **寫**：加鎖（JDK8 `ReentrantLock`，JDK9+ `synchronized(lock)`）→ `Arrays.copyOf` 複製新數組 → 喺新數組改 → `setArray` 換掉 **volatile** 引用。
- **讀**：完全唔加鎖，直接讀 volatile 數組 → 讀極快。
- **迭代器係快照**：唔會 CME（對比 ArrayList fail-fast），但睇唔到迭代開始後嘅修改；`iterator.remove()` 拋 `UnsupportedOperationException`。
- **代價**：每次寫 O(n) 複製 + 短時間兩份內存 → 大 list 頻繁寫 = 大量垃圾、Young GC 頻繁；只係**弱一致**，唔適合要即時強一致嘅場景。
- **適合**：監聽器/觀察者列表、白名單、路由表、配置快照（極少寫、大量讀/遍歷）。
- **替代**：`Collections.synchronizedList`（寫便宜但遍歷要手動 `synchronized`）；隊列用 `ConcurrentLinkedQueue`；批量更新可以用「不可變 List + volatile/AtomicReference 整體換引用」。

## 追問 / 陷阱
- 循環 `add` 一萬次 → 每次都複製，O(n²)；應該一次 `addAll`。
- 「`if (!list.contains(x)) list.add(x)`」唔係原子 → 用 `addIfAbsent`。
- 先 `size()` 再 `get(size-1)`：中間被刪會 `IndexOutOfBoundsException`；複合操作要自己加鎖或者先拎快照。
