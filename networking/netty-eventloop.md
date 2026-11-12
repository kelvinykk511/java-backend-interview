# Netty EventLoop：永不阻塞

## 面試問題

為什麼說 Netty 的 EventLoop **絕對不能阻塞**？Handler 裡同步查庫／打 RPC 會怎樣？正確的卸貨（offload）怎麼做？

## 答法

**模型一句話**

- 一條 **EventLoop = 一條執行緒 + 一個任務佇列**，串行跑：I/O 就緒事件、pipeline handler、你 `execute` 進去的任務。
- 同一個 Channel **黏**在同一條 EventLoop 上 → 該連線狀態幾乎不用加鎖；代價是：**這條執行緒一卡，掛在上面的整批連線一起卡**。

**為什麼不能阻塞**

- EventLoop 數量通常 ≈ CPU 核數級（Worker），遠小於連線數。
- 在 Handler 裡做：同步 JDBC、同步 HTTP、`Thread.sleep`、搶分布式鎖、大 JSON 序列化、等 `Future.get()` → 等於把「I/O 多路複用執行緒」當成業務執行緒用。
- 症狀：不只「這個請求慢」，而是 **同 EventLoop 上其他連線的讀寫、心跳、超時全停**——CEX 行情推送／訂單流常見「一片用戶一起超時」。

**正確拆法**

1. Pipeline 裡只留：解碼、輕量校驗、寫回、把業務丟出去。
2. **重活／阻塞 I/O** → 獨立業務執行緒池（或虛擬執行緒池，但仍要隔離與限流）。
3. 業務做完要寫 Channel：用 `ctx.channel().eventLoop().execute(...)` 或 Netty 執行緒安全 API 跳回 EventLoop，**別在業務執行緒直接亂碰非線程安全狀態**。
4. 編解碼若極重，也可考慮獨立 executor 的 `EventExecutorGroup` 掛到特定 handler（面試提到「可以卸」即可）。

**和已有筆記的分工**

- Boss/Worker 誰接連線 → 見 [Netty 執行緒模型](netty-thread-model.md)
- BIO/NIO 對比入門 → 見 [NIO / Netty Event Loop](nio-netty-event-loop.md)
- 本篇聚焦：**永不阻塞 + 卸貨 + 跳回 EventLoop**

## 常見追問／陷阱

「加更多 Worker 就能解決 Handler 阻塞？」

→ **治標不治本**。阻塞任務會吃光更多 EventLoop，上下文切換更慘；正解是 **業務池卸貨 + 限流**。另一陷阱：在業務執行緒 `channel.writeAndFlush` 後假設「一定立刻送出」——寫入仍由 EventLoop 異步刷；要麼加 listener，要麼別在錯誤執行緒假設順序。

## 關聯

- [Netty 執行緒模型](netty-thread-model.md)
- [NIO / Netty Event Loop](nio-netty-event-loop.md)
- [HTTP Keep-Alive 與連線池](http-keepalive-connection-pool.md)

---

## 深入（2026-11-12）：`ctx.executor()` 唔係卸貨，同寫返 EventLoop

### 面試可以點追

「Handler 裡 `ctx.executor().execute(() -> { 查庫; writeAndFlush; })` 算唔算已經卸貨？業務線程寫完 Channel 要注意什麼？」

### 答法

**`ctx.executor()` 就係呢條 EventLoop**

- `ChannelHandlerContext.executor()` 通常返回 **綁定呢個 Channel 嘅 EventLoop**（除非你顯式把 handler 掛到獨立 `EventExecutorGroup`）。
- 所以 `ctx.executor().execute(task)` 只係把任務排進**同一條** EventLoop 隊列尾，**下一個 tick 仍然喺 I/O 線程跑**。查庫、RPC、`Future.get()` 寫喺入面，阻塞效果同直接喺 `channelRead` 寫一樣。
- 口條：**卸貨要丟去你自己建嘅業務池**（或明確配置嘅 `EventExecutorGroup`），唔係 `ctx.executor()`。

**寫返 Channel：跳回 EventLoop**

- Channel 同 pipeline 狀態幾乎假設單線程。業務池做完要 `writeAndFlush`：用 `ctx.channel().eventLoop().execute(() -> ctx.writeAndFlush(...))`（或 Netty 標明線程安全嘅 write API），**跳返**原 EventLoop。
- 好處：同一條連線上嘅 snapshot → 成交 → 撤單，按你提交回 EventLoop 嘅次序串行寫出，唔會喺業務池亂序互踩。
- 業務線程直接 `writeAndFlush` 之後當「已經送出」係陷阱——寫入仍由 EventLoop **異步刷**；要確認就加 listener，要順序就跳返去排隊。

**阻塞時外表係「心跳死咗」**

- `IdleStateHandler`、超時、讀寫都排喺同一條 EventLoop。線程卡喺同步 I/O，心跳任務排喺後面跑唔到 → 對端當死連線、成片用戶一齊斷。
- 監控：EventLoop `pendingTask`／任務延遲（唔係只睇 JVM CPU）；心跳超時要同業務延遲分開告警。

### 追問／陷阱

「加多幾條 Worker 就得？」→ 治標不治本，見上文。另一個常見疊加：喺 EventLoop 上對業務池 `submit().get()`——I/O 線程自己堵住等業務；若業務池拒絕策略係 `CallerRuns`，任務可能**走返 EventLoop 跑**，阻塞再放大。
