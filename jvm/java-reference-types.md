# Java 四種引用：強 / 軟 / 弱 / 虛

## 面試題
強、軟、弱、虛引用分別幾時被回收？實際喺邊度用到？

## 答案要點
- **強引用**：一般 `new` 出嚟嘅引用。只要仲可達就唔會收，寧願 OOM。
- **軟引用 `SoftReference`**：內存唔夠（準備 OOM 之前）先清。HotSpot 實際按 `-XX:SoftRefLRUPolicyMSPerMB`（預設每 MB 空閒堆保留約 1 秒）決定。
  - 理論上適合做緩存，但**生產唔推薦**：回收時機唔可控，GC 壓力大時可能一次過清晒 → 緩存雪崩，仲會令 Full GC 更頻繁。用 Caffeine/Guava 嘅 `maximumSize` + 過期時間更穩陣。
- **弱引用 `WeakReference`**：冇強引用時，**下一次 GC 就收**（唔理內存夠唔夠）。
  - `WeakHashMap` 嘅 key、`ThreadLocalMap.Entry` 嘅 key 都係弱引用。
- **虛引用 `PhantomReference`**：`get()` 永遠返回 null，唯一用途係配 `ReferenceQueue`，喺對象被回收後收到通知做清理。
  - 取代已廢棄嘅 `finalize()`；Java 9+ 用 `java.lang.ref.Cleaner`。
  - `DirectByteBuffer` 靠 Cleaner 釋放堆外內存 → 堆外內存要等 GC 先釋放，所以要設 `-XX:MaxDirectMemorySize`。
- **`ReferenceQueue`**：引用指向嘅對象被回收後，引用對象本身會入隊，方便清走 map 入面已失效嘅 entry（`WeakHashMap.expungeStaleEntries` 就係咁做）。

## 常見追問 / 陷阱
- **ThreadLocal key 係弱引用，點解仲會洩漏？**
  - ThreadLocal 對象被回收後 entry 嘅 key 變 null，但 **value 仍然被強引用**：`Thread → ThreadLocalMap → Entry → value`。
  - 線程池線程長命 → value 一直喺度：內存洩漏 + 下一個請求讀到上一個用戶嘅數據。
  - 解法：`try { ... } finally { threadLocal.remove(); }`。詳見 [ThreadLocal / TTL](../concurrency/threadlocal-ttl.md)。
- **WeakHashMap 唔清嘅情況**：value 反過嚟強引用 key；或者用字串字面量做 key（常量池一直強引用住）。
- 軟引用緩存大到某個程度，可能導致 GC 後老年代仍然好滿 → 連續 Full GC。
