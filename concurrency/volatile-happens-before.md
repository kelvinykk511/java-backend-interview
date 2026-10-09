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
