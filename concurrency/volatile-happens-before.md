# volatile 與 happens-before

## 面試問題

`volatile` 能保證什麼？不能保證什麼？跟 `synchronized` 差在哪？

## 答法

`volatile` 保證兩件事：

- **可見性**：寫入立刻對其他執行緒可見（透過記憶體屏障，刷新工作記憶體）
- **有序性（禁止重排序）**：對該變數的讀寫不會被亂序到屏障另一側

**不保證原子性**：`i++` 仍是讀-改-寫三步，多執行緒會丟更新。計數要用 `AtomicInteger` / 加鎖。

典型用法：

- 狀態旗標：`volatile boolean running`
- 雙重檢查單例的 instance（JDK5+ 需要 volatile 擋住半初始化）
- 一寫多讀、寫本身已是原子賦值的場景

跟 `synchronized`：鎖同時給可見性 + 原子性 + 互斥；`volatile` 更輕，但沒有互斥。

## 常見追問 / 陷阱

「happens-before 是什麼？volatile 寫跟後續讀有什麼關係？」

→ happens-before 是 JMM 的偏序規則：A hb B 則 A 的結果對 B 可見。規則含：程式順序、鎖解鎖→加鎖、**volatile 寫 → 後續對同一變數的讀**、執行緒 start/join、傳遞性。面試講清「可見性來自 hb，不是魔法」較加分。
