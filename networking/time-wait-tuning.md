# TCP TIME_WAIT 調優

> 狀態含義同 CLOSE_WAIT 對照見 [TIME_WAIT 與 CLOSE_WAIT](time-wait-close-wait.md)

## 面試題

一台調用下游 HTTP 服務嘅 Java 機，`ss -s` 見到幾萬個 TIME_WAIT，仲開始報 `Cannot assign requested address`，點解？點調？

## 標準答案

### TIME_WAIT 係咩

- 喺**主動關閉方**；Linux 固定停 **60s**（2MSL 概念）。
- 作用：最後一個 ACK 丟咗可以重發；防舊連接遲到嘅包混入新連接（同四元組）。

### 問題本質

- **客戶端出站端口耗盡**。
- 四元組 `(src ip, src port, dst ip, dst port)` 對同一個下游只有 **src port** 會變。
- `ip_local_port_range` 預設 `32768–60999` 約 **2.8 萬**個；60s 內對同一 dst 短連接超過約 **470 QPS** 就可能耗盡。

### 根治

- **長連接 + 連接池**：[HTTP Keep-Alive 與連線池](http-keepalive-connection-pool.md)（Apache HttpClient / OkHttp、`maxPerRoute` 合理）。
- 唔好每次請求 new client；**response body 要讀完／close** 先可以復用。

### 系統參數（治標）

| 參數 | 說明 |
|------|------|
| `net.ipv4.ip_local_port_range` | 擴大（例如 `1024 65535`，避開服務監聽端口） |
| `net.ipv4.tcp_tw_reuse=1` | 只對**主動 connect** 嘅出站連接生效，靠 TCP timestamps 判新舊，相對安全 |
| `net.ipv4.tcp_max_tw_buckets` | TIME_WAIT 上限，超出會直接銷毀並寫 log |

- **唔好用 `tcp_tw_recycle`**：NAT 後面多客戶端會因 timestamp 唔單調被丟包，Linux 4.12 已移除。

### 服務端大量 TIME_WAIT

- 通常係服務端**主動關**（Nginx 對 upstream 冇開 keepalive、`Connection: close`、HTTP/1.0）。
- Nginx upstream 開 keepalive + `proxy_http_version 1.1` + 清空 `Connection` header。

### 分清 CLOSE_WAIT

- 大量 **CLOSE_WAIT** = 自己程式收到 FIN 但冇 `close()`（代碼 bug／連接洩漏），同 TIME_WAIT **完全唔同**處理方法 → 見 [TIME_WAIT 與 CLOSE_WAIT](time-wait-close-wait.md)。

## 追問・陷阱

- **`SO_REUSEADDR` 可唔可以解決？** 主要令 server 重啟時可以 bind 返仍有 TIME_WAIT 嘅**監聽端口**，解決唔到客戶端出站端口耗盡。
- **`SO_LINGER` timeout=0** 直接發 RST 跳過 TIME_WAIT：會丟未送完數據，唔應該當常規手段。

## 實戰 (CEX) 角度

- 行情／風控／錢包服務之間**高頻 HTTP 調用**，一定要用**共享連接池**。
- 監控 TIME_WAIT 數量同連接池 **pending** 數，同下游 SLA 一齊睇。

## See also

- [TIME_WAIT 與 CLOSE_WAIT](time-wait-close-wait.md)
- [HTTP Keep-Alive 與連線池](http-keepalive-connection-pool.md)
- [HTTP Client 超時配置](http-client-timeouts.md)
