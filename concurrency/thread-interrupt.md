# 線程中斷 interrupt：點樣正確停止一條線程

## 面試問題

點樣停止一條正在跑嘅 Java 線程？`interrupt()` 係咪會即刻殺死線程？

## 答法

- **`Thread.stop()` 已廢棄**：會喺任意位置拋 `ThreadDeath`、釋放所有鎖，數據可能改到一半 → 唔一致。JDK 20+ 直接拋 `UnsupportedOperationException`。
- **`interrupt()` 只係「舉手示意」**：設一個中斷標誌，線程要**自己配合**先會停。
  - 線程喺 `sleep` / `wait` / `join` / `BlockingQueue.take` / `Lock.lockInterruptibly` 等阻塞中 → 拋 `InterruptedException`，**同時清除標誌**。
  - 線程喺普通計算中 → 乜都唔會發生，要自己檢查 `Thread.currentThread().isInterrupted()`。
- **三個 API 分清楚**：
  - `t.interrupt()`：設標誌。
  - `t.isInterrupted()`：查標誌，唔清除。
  - `Thread.interrupted()`（static）：查**並清除**當前線程標誌 —— 好易誤用。
- **標準寫法**：
  ```java
  while (!Thread.currentThread().isInterrupted()) {
      try {
          Task t = queue.take();
          handle(t);
      } catch (InterruptedException e) {
          Thread.currentThread().interrupt(); // 恢復標誌
          break;                               // 收尾退出
      }
  }
  ```
- **線程池**：`shutdown()` 唔收新任務、等舊任務做完；`shutdownNow()` 對所有 worker 調 `interrupt()` 並返回未執行任務；`Future.cancel(true)` 都係靠 interrupt。任務唔配合中斷 → 停唔到。

## 常見追問 / 陷阱

- **最常見 bug：食咗 InterruptedException**：`catch (InterruptedException e) { log.warn(...); }` 之後標誌已被清除，外層 `while (!isInterrupted())` 永遠 true → `shutdownNow()` 停唔到、應用優雅停機卡住。要麼往上拋，要麼 `Thread.currentThread().interrupt()` 恢復。
- 唔好用 `Thread.interrupted()` 做循環條件判斷再唔處理，會偷偷清咗標誌。
- `synchronized` 等鎖**唔響應中斷**；要可中斷就用 `ReentrantLock.lockInterruptibly()`。
- 業務層亦可以用 `volatile boolean running` 做停止旗，但阻塞喺 `take()` 時睇唔到旗，通常要配 interrupt 一齊用。

## 一句話

interrupt 係協作式：設標誌 + 阻塞方法拋異常；catch 到要恢復標誌或者退出，唔好食咗佢。

---

## 補充（2026-11-02 drill）：BIO Socket read 唔響應中斷

- 傳統 `Socket.getInputStream().read()` 唔係可中斷 I/O，`interrupt()` 只設標誌，線程照卡。
- 解法：`setSoTimeout` + loop 檢查中斷標誌；停機時另一線程 `socket.close()`（read 拋 `SocketException`）；或改用 NIO `SocketChannel`（`ClosedByInterruptException`）。
- JDK 21 虛擬線程例外：阻塞喺 Socket I/O 時被 interrupt 會關 socket 拋 `SocketException`。
