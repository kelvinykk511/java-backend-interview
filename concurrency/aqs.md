# AQS（AbstractQueuedSynchronizer）

## 面試問題

`ReentrantLock`、`Semaphore`、`CountDownLatch` 底層都是 AQS，AQS 到底做了什麼？公平鎖和非公平鎖差在哪？

## 答法

**一句話**：AQS = 一個 `volatile int state` + 一條 CLH 變體的**雙向等待隊列**；子類只定義「state 怎麼算搶到」，排隊、掛起、喚醒交給 AQS。

**三件套**

- **state**：用 CAS 改。`ReentrantLock` 用它記重入次數；`Semaphore` 記剩餘許可；`CountDownLatch` 記剩餘計數。
- **隊列**：搶不到的線程包成 Node 入隊尾（CAS 接尾），然後 `LockSupport.park()` 掛起。
- **喚醒**：釋放時改 state，再 `unpark` 隊頭的下一個節點，它醒來**再嘗試一次** CAS。

**獨佔 vs 共享**

- 獨佔（`tryAcquire/tryRelease`）：`ReentrantLock`。
- 共享（`tryAcquireShared/tryReleaseShared`）：`Semaphore`、`CountDownLatch`、讀鎖；釋放會向後**傳播**喚醒。

**公平 vs 非公平（ReentrantLock 預設非公平）**

- 非公平：新來的線程先直接 CAS 搶一次，搶到就插隊。好處：少一次 park/unpark 上下文切換，吞吐高。代價：隊列裡的可能餓。
- 公平：先看 `hasQueuedPredecessors()`，有人排隊就乖乖排。順序好、吞吐低。

**Condition**

- `lock.newCondition()` 是另一條**條件隊列**；`await()` 釋放鎖進條件隊列，`signal()` 把節點搬回同步隊列再搶鎖。比 `wait/notify` 好在可以多個條件分開喚醒（例如「非空」「非滿」兩個隊列）。

## 常見追問 / 陷阱

「`synchronized` 和 AQS 鎖都會阻塞，差在哪？」

→ `synchronized` 是 JVM 內建 monitor（有鎖升級），自動釋放；AQS 鎖是 Java 代碼 + CAS + park，**可中斷（`lockInterruptibly`）、可超時（`tryLock(timeout)`）、可公平、多 Condition**，但必須 `finally { unlock(); }`。陷阱：`unlock` 漏寫或寫錯位置 → 鎖永遠不釋放，jstack 看到一堆 `WAITING (parking)` 在 `AbstractQueuedSynchronizer`。

## 小練習

撮合前的風控檢查用了公平 `ReentrantLock`，壓測吞吐只有非公平的一半——為什麼？如果業務要求「不能讓某個帳戶的請求一直搶不到」，除了公平鎖還有什麼做法（提示：按帳戶分片串行化）？
