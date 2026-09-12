# HTTP Keep-Alive 與連線池

## 面試問題

HTTP Keep-Alive 解決什麼問題？後端用 HttpClient／連線池時，常見坑有哪些（超時、連線洩漏、對端關連線）？

## 模範答案（面試可講）

- HTTP/1.0 預設短連線：每次請求 TCP 握手（+ 可能 TLS）成本高。
- **Keep-Alive（持久連線）**：同一條 TCP 上跑多個 HTTP 請求，省握手、減延遲。
- HTTP/1.1 預設 `Connection: keep-alive`；HTTP/2 進一步用**多路複用**，一條連線並行多個 stream。
- **連線池**：客戶端複用空閒連線；關鍵參數大致是：最大連線數、每路由上限、連線／讀取／寫入超時、空閒驅逐。
- 實務坑：
  - **沒消費完 response body** → 連線無法歸還池（洩漏）。
  - **對端已關閉**，池裡還以為可用 → 下一次才爆；要設驗證／重試策略。
  - **超時只設 connect、不設 socket/read** → 慢對端拖死執行緒。
  - 池太小 → 排隊；太大 → 壓垮對端或本機 fd。

## 常見追問／陷阱

- Keep-Alive ≠ 業務長輪詢；也別把「開了 Keep-Alive」講成「一定比 HTTP/2 快」。
- 服務端也有 keep-alive timeout；兩邊不一致時會出現偶發 `Connection reset`。

## 關聯

- [TCP 三次握手](tcp-three-way-handshake.md)
