# CAS、ABA 與 LongAdder

## 面試問題

講清楚 CAS 點運作、ABA 幾時危險、LongAdder 同 `AtomicLong` 點揀？

## 答法

**CAS（Compare-And-Swap）**

- CPU 指令級「期望值一致先更新」，失敗就**自旋重試**。
- 高競爭時：CPU 燒、**cache line 喺核之間來回跳**（false sharing 會雪上加霜）。
- `AtomicLong` = **單一熱點變數**，所有線程打同一個字。

**LongAdder**

- 結構：`base` + `Cell[]`；無競爭只寫 `base`，有競爭按線程 hash 落不同 `Cell`。
- `sum()` = `base` + 所有 `Cell`；`Cell` 用 `@Contended` 減 false sharing。
- **代價**：`sum()` **唔係**原子快照（讀取過程中可能仲有寫入）；記憶體多過一個 `AtomicLong`。
- **適用**：QPS、成交筆數等**近似統計**；**唔適用**：訂單號、庫存扣減等要**精確、可重複讀同一數值**嘅場景 → 繼續用 `AtomicLong` 或鎖。

**ABA**

- 值 A → B → A，CAS 只比「當前值係咪 A」，**睇唔到中間改過**。
- 純數字加減多數無害；**指針 / 鏈表**（例如 lock-free stack pop）會用返已釋放節點 → 嚴重 bug。
- 解法：`AtomicStampedReference`（值 + 版本戳）；DB 用 **version 欄** 做樂觀鎖。

**CAS 局限（追問用）**

- ABA（結構性數據）
- 自旋成本（高競爭）
- **單變數**原子；多欄位一致要 `AtomicReference` 裝快照、或直接用鎖

## 常見追問 / 陷阱

「LongAdder 可以完全取代 AtomicLong？」

→ **唔可以**。要全局一致讀數、CAS 迴圈做業務判斷（例如 `compareAndSet` 限庫存）必須用 `AtomicLong` / `AtomicInteger`；LongAdder 只適合「只加、偶爾讀近似總和」嘅 metrics。

## 小練習

Prometheus 每秒 export 一次「全站成交筆數」，同時另一條線程用 CAS 派全局遞增訂單號。兩個計數器你會點揀？理由係咩？
