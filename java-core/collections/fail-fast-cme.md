# 集合 fail-fast 與 ConcurrentModificationException

## 面試問題

`for (String s : list) { if (...) list.remove(s); }` 為什麼會拋 `ConcurrentModificationException`（CME）？單線程也會嗎？正確寫法有哪些？

## 答法

**機制一句話**：`ArrayList`/`HashMap` 有個 `modCount`（結構修改次數）。迭代器建立時記下 `expectedModCount`，每次 `next()` 都對比；你繞過迭代器直接 `list.remove()`，`modCount` 變了 → 下一次 `next()` 發現對不上 → 拋 CME。

- **單線程也會**：CME 跟「並發」無關，只看「迭代期間有沒有繞過迭代器做結構修改」。
- **fail-fast 是盡力而為**：只是檢測 bug 的保險，**不是線程安全保證**；多線程下可能不拋、直接讀到髒數據。
- **怪現象**：刪「倒數第二個」元素時，`hasNext()` 剛好判斷 `cursor == size` 返回 false，循環直接結束，**不拋異常但漏掉最後一個**——面試官喜歡考。

**正確寫法**

1. `Iterator.remove()`：會同步 `expectedModCount`。
2. `list.removeIf(x -> ...)`（Java 8+，首選，內部批量處理，ArrayList 下是 O(n)）。
3. 倒序下標 `for (int i = size-1; i >= 0; i--) list.remove(i)`。
4. 收集後統一刪 / 用 Stream filter 生成新集合。

**fail-safe（快照/弱一致）對比**

- `CopyOnWriteArrayList`：迭代的是**快照**，不拋 CME，但寫時複製整個數組 → 讀多寫極少才用（例如監聽器列表、配置）。
- `ConcurrentHashMap`：迭代器**弱一致**，不拋 CME，可能看到也可能看不到迭代期間的修改。

## 常見追問 / 陷阱

「把 `ArrayList` 換成 `Collections.synchronizedList` 就不會 CME 了嗎？」→ **不會**。它只鎖單個方法，迭代過程不是原子的，仍要手動 `synchronized (list) { for ... }`。陷阱二：在 `HashMap` 的 `forEach`/`entrySet` 迭代裡 `put` 新 key 一樣會 CME；改值（`entry.setValue`）不算結構修改，不會。

## 小練習

訂單撮合後要從 `List<Order>` 裡移除已成交訂單，同事用增強 for + `remove`，測試數據剛好只刪倒數第二個所以沒報錯，上線後偶爾 CME。解釋為什麼測試沒發現，並給出兩種改法。
