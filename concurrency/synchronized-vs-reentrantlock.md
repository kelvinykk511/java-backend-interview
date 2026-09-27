# synchronized vs ReentrantLock

## 面試問題

`synchronized` 和 `ReentrantLock` 怎麼選？底層大致差在哪？

## 模範答案（面試官向）

- **相同點**：都可重入；都能保證互斥與可見性（進出臨界區建立 happens-before）。
- **`synchronized`（JVM 內建）**：
  - 語法簡單，異常路徑也會自動釋放鎖。
  - 早期偏重量級；現代 HotSpot 有偏向鎖／輕量鎖／膨脹到重量級（具體隨版本演進，面試講「有鎖升級／自适应」即可）。
  - **不能**：嘗試鎖失敗就放棄、限時等鎖、被中斷地等鎖、一把鎖配多個獨立條件隊列。
- **`ReentrantLock`（JUC，AQS）**：
  - 必須在 `finally` 裡 `unlock`，忘了就死鎖。
  - **可中斷**（`lockInterruptibly`）、**可超時**（`tryLock(timeout)`）、可選**公平鎖**。
  - 一把鎖可建多個 `Condition`（類似多個 wait set），適合複雜協調。
- **怎麼選**：臨界區單純、鎖持有短 → `synchronized` 夠用且更不容易寫錯；需要超時／中斷／多條件／公平性 → 上 `ReentrantLock`。

## 常見追問／陷阱

- 「公平鎖一定更好？」→ 公平鎖吞吐通常更差（減少插隊但增加調度開銷），預設**非公平**就好，除非明確要防飢餓。
- 「`synchronized` 不能中斷等待」→ 準確說：在 `synchronized` 上阻塞的線程**不能**像 `lockInterruptibly` 那樣響應中斷而退出等鎖；被中斷只是打標記，仍會繼續等鎖。
- 別和 `volatile` 搞混：`volatile` 只保證可見／禁止重排，**不**提供互斥複合操作（如 check-then-act）。
