# volatile 與 happens-before（JMM）

## 面試問題

`volatile` 能保證什麼、不能保證什麼？為什麼雙重檢查單例（DCL）的 `instance` 一定要 `volatile`？

## 答法

`volatile` 保證兩件事（JMM）：

- **可見性**：寫入對後續其他執行緒的讀可見（經 memory barrier / store-load 語義）
- **有序性**：禁止把該變數的讀寫重排序到屏障另一側

**不保證原子性**：`i++` 仍是讀-改-寫三步，多執行緒會丟更新 → 用 `AtomicInteger` 或加鎖。

典型用法：

- 狀態旗標：`volatile boolean running`（一寫多讀、賦值本身原子）
- DCL 單例的 `instance`（JDK5+）：擋住「引用先可見、構造尚未完成」的半初始化
- 一寫多讀、寫是單一原子賦值的場景

跟 `synchronized`：鎖 = 可見性 + 原子性 + 互斥；`volatile` 更輕，但沒有互斥。

DCL 為什麼要 `volatile`：

1. `new Singleton()` 可拆成：配記憶體 → 初始化欄位 → 把引用賦給 `instance`
2. 無 `volatile` 時，2 與 3 可能重排序；其他執行緒看到非 null 引用，但欄位還是預設值 → 奇異 bug
3. `volatile` 寫建立 happens-before，擋住這類重排序

## 常見追問 / 陷阱

「happens-before 是什麼？`volatile` 寫跟後續讀有什麼關係？」

→ happens-before 是 JMM 偏序：A hb B 則 A 的結果對 B 可見。常見規則：程式順序、解鎖 → 加鎖、**volatile 寫 → 後續同一變數的讀**、thread start/join、傳遞性。面試講「可見性來自 hb，不是魔法」較加分。

陷阱：`long` / `double` 非 volatile 時，64-bit 寫在部分 JVM 上可能非原子（撕裂讀）；要可見又原子才用 `volatile` / `AtomicLong`。`volatile` 陣列只保護引用本身，不保護元素。


## 深入：安全發布 vs 複合操作（2026-11-26 warm-up）

**面試題**：`boolean running`（冇 volatile）做停止旗標，worker `while (running)` 停唔到。加咗 volatile 之後，有人「先改 Config 各欄位，再把 `volatile Config published` 指向新對象」；另一處用 `volatile int seq; seq++` 派單號。咁做安全嗎？`volatile` 同 `synchronized` 點分？

- **可見性來自 happens-before，唔係魔法**。冇 `volatile`／鎖時，JIT 可以把 `running` 提升到暫存器、CPU 亦可以只睇自己 cache，worker 會一直見到 `true`。`volatile` 寫 happens-before 其後另一條線程對**同一變量**嘅讀，寫入先對讀可見，亦禁止把該變量嘅讀寫重排到屏障另一側。
- **安全發布（safe publication）成立嘅條件**：先把新對象各欄位寫完，**最後**先做 `volatile` 引用賦值。程序順序令「欄位寫 happens-before volatile 寫」；另一條線程 volatile 讀 happens-before 再讀欄位。傳遞性之下，讀到新引用就會見到初始化完成嘅欄位。呢個同 DCL 要把 `instance` 標成 `volatile` 係同一條原則。
- **發布完之後再改就唔再保證**。若讀線程拿嘅係另一個**非 volatile** 引用（例如先寫咗 `cfg = n` 再 `published = n`，loop 卻讀 `cfg`），可以一直見到舊對象或未初始化完嘅值。Config 發布後應當**不可變**；後續改動要重新發布新對象，或用鎖／volatile 欄位保護每一次寫。
- **`volatile` 唔保證複合操作原子**。`seq++` 仍係讀-改-寫三步，兩個線程會派到同一個號。要用 `AtomicInteger`／`LongAdder`，或者鎖。旗標賦值之所以 OK，係因為「寫一個 boolean／引用」本身係原子嘅。
- **同 `synchronized`**：鎖同時提供可見性、原子性同互斥；`volatile` 更輕，只適合一寫多讀嘅旗標，同「寫完一個引用就發布」。臨界區、計數、先檢查再寫，都唔能只靠 `volatile`。

### 追問

- DCL：`new Singleton()` 可拆成配記憶體 → 初始化欄位 → 把引用賦值。冇 `volatile` 時後兩步可能重排，其他線程見到非 null 但欄位仲係預設值。
- `volatile` 陣列只保護**陣列引用**本身，元素嘅讀寫冇 happens-before。`long`／`double` 非 volatile 時 JMM 允許 64-bit 撕裂讀；要可見又原子先用 `volatile` 或 `AtomicLong`。
