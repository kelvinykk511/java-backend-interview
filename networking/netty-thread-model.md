# Netty 執行緒模型（Boss / Worker 深挖）

## 面試問題

Netty 的 Boss 與 Worker 怎麼分工？為什麼 Handler 裡阻塞會拖垮一整片連線？

## 答法

**分工**

- **Boss EventLoopGroup**（常 1 條執行緒）：只負責 `accept` 新連線，把 Channel 註冊到 Worker。
- **Worker EventLoopGroup**（N 條，常 ≈ CPU 核數 × 2）：每條 EventLoop **固定**管一批 Channel 的全程 I/O（讀寫、事件、pipeline）。
- **執行緒親和**：同一個 Channel 的入站事件幾乎總在同一條 EventLoop 上跑 → 無鎖串行處理該連線狀態，這是模型的精華。

**為什麼不能阻塞 EventLoop**

- 一條 EventLoop 卡住（同步 JDBC、大計算、`sleep`、搶遠端鎖）→ 掛在它上面的 **所有 Channel** 的 I/O 與超時都停擺。
- 正確拆法：Pipeline 裡 I/O 與編解碼留在 EventLoop；業務丟 **獨立業務執行緒池**（`ctx.channel().eventLoop().execute` 只丟回輕邏輯；重活用 `group`/`executor`）。

**和「一請求一執行緒」比**：連線數 ≫ 執行緒數；吞吐靠非阻塞 I/O + 短任務，不是靠開幾千條執行緒。

## 常見追問 / 陷阱

「Worker 設越大越好？」

→ 不是。過多 EventLoop 增加切換與緩存不命中；業務阻塞更該加 **業務池**，不是盲目加 Worker。另一陷阱：在非 EventLoop 執行緒直接碰不是 thread-safe 的 Channel 狀態 —— 跨執行緒操作要用 `eventLoop().execute/submit` 或 Netty 提供的安全 API。

## 小練習

gRPC/Netty 服務裡 Handler 同步調下游 HTTP 3 秒超時，線上表現會是「單連線慢」還是「一片連線一起卡」？你會怎麼改？
