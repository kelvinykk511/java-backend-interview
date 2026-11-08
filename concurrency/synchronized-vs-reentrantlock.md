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

## 深入（2026-11-08）：鎖升級之後點揀

### 面試問題

而家都知 `synchronized` 有鎖升級，JDK 15+ 仲預設關閉偏向鎖，仲有必要用 `ReentrantLock` 嗎？點揀？

### 答法

- **性能**：無競爭、短臨界區時，現代 HotSpot 嘅 `synchronized`（輕量級鎖 + 自適應自旋）同 `ReentrantLock`（AQS + CAS）都好快，唔好為「顯得高級」而換。兩邊膨脹成重量級之後都係 OS park／unpark，吞吐接近。詳見 [鎖升級](synchronized-lock-upgrade.md)。
- **JDK 15+（JEP 374）** 預設關並棄用偏向鎖：少咗「單線程反覆進同一把鎖」嘅微優勢，亦少咗撤銷偏向要進 safepoint 嘅坑。面試唔好仲把 JDK 8 偏向鎖當唯一主線。
- `synchronized` 仍然贏在：語法簡單、異常路徑自動釋放、JIT 鎖消除／鎖粗化更典型。臨界區單純、持鎖短 → 繼續用佢。
- **只有 `ReentrantLock` 做得到**（呢啲先係換鎖嘅理由）：
  - `tryLock`／`tryLock(timeout)`：搶唔到就快速失敗／降級，避免死等。
  - `lockInterruptibly`：停機或取消任務時可以離開等鎖。
  - 可選公平鎖（少數要防飢餓；預設非公平吞吐更好）。
  - 一把鎖配多個 `Condition`（多個等待隊列，適合複雜協調）。
- **虛擬線程（JDK 21）**：喺 `synchronized` 入面做阻塞 IO 會 **pin** 住 carrier；`ReentrantLock` 嘅 park 可以 unmount。臨界區內可能打 RPC／寫 Redis 時，JDK 21 優先 `ReentrantLock`，或者把 IO 移出鎖。JDK 24（JEP 491）先解決 synchronized pin。
- 口訣：短、純記憶體操作、自動釋放 → `synchronized`；要超時／中斷／多條件／公平，或 JDK 21 虛擬線程下鎖內可能阻塞 → `ReentrantLock`。用 `ReentrantLock` 必須 `finally` 裡 `unlock`。

### 追問／陷阱

- 「鎖升級之後兩邊完全等價」→ **性能近，API 能力唔等價**。
- 把整段遠程調用包進任何一種鎖都係災難：持鎖時間變長，競爭同超時會一齊爆。先縮短臨界區，再談用邊把鎖。
- 公平 `ReentrantLock` 唔應當預設——減少插隊但調度更貴，吞吐通常更差。
