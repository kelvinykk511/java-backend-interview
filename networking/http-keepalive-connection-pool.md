# HTTP Keep-Alive 與連線池

## 面試問題

HTTP Keep-Alive 解決什麼問題？後端用 HttpClient／連線池時，常見坑有哪些（超時、連線洩漏、對端關連線）？高併發下池子怎麼調？

## 模範答案（面試可講）

- HTTP/1.0 預設短連線：每次請求 TCP 握手（+ 可能 TLS）成本高。
- **Keep-Alive（持久連線）**：同一條 TCP 上跑多個 HTTP 請求，省握手、減延遲。
- HTTP/1.1 預設 `Connection: keep-alive`；HTTP/2 進一步用**多路複用**，一條連線並行多個 stream。
- **連線池**：客戶端複用空閒連線；關鍵參數大致是：最大連線數、每路由上限、連線／讀取／寫入超時、空閒驅逐、從池取連線的等待超時。

### 實務坑

- **沒消費完 response body** → 連線無法歸還池（洩漏）→ 池耗盡 → 新請求卡死或超時。
- **對端已關閉**，池裡還以為可用 → 下一次才爆；要設驗證／重試策略（或短一點 idle timeout）。
- **超時只設 connect、不設 socket/read** → 慢對端拖死執行緒。
- **池太小** → 高併發排隊（看起來像下游慢，其實是自己在等連線）；**太大** → 壓垮對端或本機 fd／執行緒。
- **服務端 keep-alive timeout 比客戶端短** → 偶發 `Connection reset`／`NoHttpResponseException`；客戶端應允許對「可重試的幂等請求」做一次重試。

### 高負載口訣（CEX 調下游常見）

1. 先看指標：池活躍／等待獲取連線數／獲取超時次數，別只看 P99 latency。
2. 每路由上限 ≈「單下游實例能扛的並發」；總上限 ≈ 路由數 × 每路由。
3. 讀超時對齊 SLA；獲取連線超時要**短於**業務超時，否則業務先爆、池還在傻等。
4. 務必 try-with-resources／保證 entity 被 consume，否則洩漏比調參更致命。

## 常見追問／陷阱

- Keep-Alive ≠ 業務長輪詢；也別把「開了 Keep-Alive」講成「一定比 HTTP/2 快」。
- 服務端也有 keep-alive timeout；兩邊不一致時會出現偶發 `Connection reset`。
- 「加線程就能解決連線池排隊？」→ 往往更糟：更多線程搶同一池，排隊變長、下游被打爆。

## 關聯

- [TCP 三次握手](tcp-three-way-handshake.md)
