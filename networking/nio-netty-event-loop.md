# Java NIO / Netty Event Loop（面試）

## 面試問題

BIO、NIO、Netty 的 EventLoop 差在哪？後端服務為什麼常選 Netty 而不是「一請求一執行緒」？

## 答法

- **BIO**：一條連線（或一個請求）佔一個執行緒，阻塞在 `read`/`accept`。連線多 → 執行緒多 → 上下文切換與記憶體爆。
- **NIO（Java）**：`Selector` + 非阻塞 `Channel`；少數執行緒輪詢「哪些 channel 就緒」，再處理 I/O。核心是 **I/O 多路複用**，不是「自動變快」。
- **Netty**：把 NIO 包成好用的模型：
  - **EventLoop**：一條執行緒跑一個（或一組）loop，負責註冊在它上面的 Channel 的所有 I/O 事件。
  - **EventLoopGroup**：boss 接連線、worker 處理已建立連線（常見分工）。
  - **Pipeline + Handler**：入站/出站責任鏈；業務邏輯別阻塞 EventLoop。
- 面試一句話：用少量執行緒處理大量連線；**EventLoop 執行緒絕不能做重計算或同步阻塞**，否則整組 Channel 卡死。

## 常見追問 / 陷阱

「NIO 一定比 BIO 快？」

→ 不一定。連線少、業務全是 CPU 重活時，BIO 可能更單純。NIO/Netty 贏在 **高並發連線 + 大量空閒等待**。另一陷阱：在 Handler 裡 `Thread.sleep`、同步查庫、搶鎖 → 等於把 EventLoop 當業務執行緒用，會拖垮吞吐。重活丟到業務執行緒池。
