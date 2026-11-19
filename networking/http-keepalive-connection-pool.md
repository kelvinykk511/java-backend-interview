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
- [HTTP 客戶端超時](http-client-timeouts.md)

---

## 深入（2026-11-19）：開咗 Keep-Alive 仍然超時、偶發 reset

### 面試可以點追

下單同步調風控／行情 HTTP，已經開咗 Keep-Alive 同連線池，但高峰期大量請求卡在「等連線」、Tomcat 線程升滿，仲偶發 `NoHttpResponseException`／`Connection reset by peer`。有人建議把池同線程再加大。點排查？Keep-Alive 同 HTTP/2 多路複用有咩分別？

### 答法

**Keep-Alive 解決咩**

- 同一條 TCP（+ 已完成嘅 TLS）上跑多個 HTTP 請求，省重複握手。HTTP/1.1 預設持久連線，但**一條連線同一時間通常只跑一個請求**（要等上一個 response 完）。
- HTTP/2 係**一條連線上多路複用多個 stream**，唔係「更長嘅 Keep-Alive」。Keep-Alive 開咗，唔等於一定快過 HTTP/2；協議、頭部壓縮、隊頭阻塞都唔同。

**卡在「等連線」常見原因（先查指標，唔好先加線程）**

- **每次 `new HttpClient()`／`HttpClients.createDefault()`**：根本冇複用，Keep-Alive 形同虛設，每次都完整握手，端口同 CPU 都被握手打滿。客戶端要單例／進程內共享一個池。
- **Response body 沒消費完**（讀完 header 就 return、異常路徑漏咗 close）：連線無法歸還，池表面仲有名額，實際可用為 0。必須 try-with-resources／保證 entity 被 consume。
- **池每路由上限太細**：高併發全部喺 pool acquire 排隊，延遲飆高看起來似下游慢。對齊「單下游能扛嘅並發」，並設**短過業務超時**嘅 acquire timeout，避免 Tomcat 線程無限卡死。
- **用加 Tomcat 線程去「解決」排隊**：更多線程搶同一個池，等待更長，下游更容易被打爆。要限流、拆池（風控／行情分開），唔係加線程。

**偶發 reset／`NoHttpResponseException`**

- 服務端 keep-alive timeout **短過**客戶端 idle 持有時間：對端已經關，池裡仲以為可用，下次取出先爆。客戶端 idle 驅逐要**短過**對端；可開取出時驗證，但唔好每個請求都做一次完整往返。
- 對端重啟、中間 LB／NAT idle 切斷，同一類症狀。
- 冪等 GET 可以對呢類「連線已死」做**一次**重試；下單／出金等非冪等請求唔好盲目重試，要靠業務冪等鍵。

**點睇**

- 池指標：leased（借出）、available、pending（等借）、acquire timeout 次數。pending 高先考慮池或洩漏，唔好只看 P99。
- 本機連線：大量新生握手、TIME_WAIT，通常係冇複用；CLOSE_WAIT 就要查有冇漏 close。

### 追問／陷阱

- Keep-Alive ≠ 業務長輪詢，亦唔等於「一定比短連線正確」——池參數同洩漏先決勝負。
- 服務端同客戶端 timeout 唔對齊，係偶發 reset 嘅高頻原因，唔好一開始就怪下游代碼。
- 連線被對端關掉之後重試，只適用冪等請求；寫操作重試可能重複下單。
