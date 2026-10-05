# TCP 四次揮手：TIME_WAIT 與 CLOSE_WAIT

## 面試問題

線上機器 `netstat` 看到大量 `TIME_WAIT` 或大量 `CLOSE_WAIT`，各代表什麼？哪個更危險？怎麼處理？

## 答法

**四次揮手一句話**：主動關閉方發 FIN → 對方 ACK → 對方處理完也發 FIN → 主動方 ACK 後進 **TIME_WAIT** 等 2MSL 才真正消失。

**TIME_WAIT（主動關閉方）**

- 為什麼要等：① 最後一個 ACK 丟了，對方重發 FIN 時還能回；② 讓舊連接的遲到包在網路裡死光，不污染同四元組的新連接。
- 大量出現的典型原因：**短連接**、客戶端頻繁主動關（例如每次 HTTP 調用都 new 連接、沒用連接池），或服務端主動關（HTTP/1.0、`Connection: close`）。
- 風險：耗盡本地臨時端口（客戶端連同一個目標 IP:port 時），報 `Cannot assign requested address`。
- 處理：**首選連接池 / keep-alive 長連接**；其次 `net.ipv4.tcp_tw_reuse=1`（客戶端側，配合 timestamps）、擴大 `ip_local_port_range`。別用已移除的 `tcp_tw_recycle`（NAT 後丟包）。

**CLOSE_WAIT（被動關閉方）**

- 含義：對方已經 FIN 了，**我方應用還沒調 `close()`**。
- 大量出現 = **代碼 bug**：連接/流沒關（漏 `finally`/try-with-resources）、HttpClient 響應 body 沒讀完沒釋放、線程卡住沒走到 close。
- 風險：不會自己消失，fd 洩漏 → `Too many open files`，連接池被佔滿。
- 處理：查代碼釋放邏輯，jstack 看卡在哪；調內核參數**治不了**。

**一句口訣**：TIME_WAIT 多是「設計/用法」問題（短連接），通常正常；CLOSE_WAIT 多是「代碼沒關」問題，一定要修。

## 常見追問 / 陷阱

「為什麼 TIME_WAIT 是 2MSL 不是 1MSL？」→ 一來一回：自己的 ACK 可能要 1MSL 才到/丟失，對方重發的 FIN 再 1MSL 回來。陷阱：說「TIME_WAIT 在服務端很多就調 `tcp_tw_reuse`」——它主要對**發起連接**的一方生效，服務端應該先想為什麼是自己在主動關。

## 小練習

行情網關調用內部報價服務，每次請求 new 一個 HttpClient 用完 close，高峰報 `Cannot assign requested address`；另一台機器 CLOSE_WAIT 持續上漲到幾萬。兩台各是什麼問題、各先改哪裡？
