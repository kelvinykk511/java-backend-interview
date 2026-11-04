# HTTPS / TLS 基礎

## 面試問題

HTTPS 比 HTTP 多了什麼？簡述 TLS 握手裡客戶端怎麼確認「連到的是真網站」？

## 答法

- **HTTPS = HTTP over TLS**：傳輸加密、完整性（防竄改）、（通常）伺服器身份認證。
- **證書鏈**：伺服器出示證書 → 客戶端用系統/內建信任的 **CA 根證書** 驗證鏈是否可信、域名是否匹配、是否過期/撤銷。
- **握手大意**（面試夠用版）：
  1. ClientHello / ServerHello（協定版本、密碼套件）
  2. 伺服器送證書（+ 可選中間證書）
  3. 金鑰交換（現代常見 ECDHE），雙方算出對稱會話金鑰
  4. 之後應用資料用對稱加密傳（AES-GCM 等）
- **為什麼不全程非對稱？** 非對稱慢；握手用非對稱/DH 建共享秘密，後續對稱加密資料。
- **面試一句話**：HTTPS 先用證書確認對方是誰，再協商出對稱金鑰加密後面的 HTTP。

## 常見追問 / 陷阱

「抓包還能看到什麼？自簽證書為什麼瀏覽器報錯？」

→ 仍可能看到 SNI 域名、IP、大致流量大小；內容與多數 header 在 TLS 內。自簽不在信任 CA 鏈上，驗證失敗——開發可暫時信任，生產要用正式 CA 或正確內網 PKI。

## 小練習

後端服務對接外部 API 時，關閉證書校驗（trust-all）有什麼風險？面試時你怎麼說「什麼時候絕對不該關」？

---

## 進階（2026-11-04）：Java 調第三方 HTTPS 報 `PKIX path building failed`

### 面試問題

一直正常嘅 Java 服務，某日調合作方 HTTPS API 開始報 `SSLHandshakeException: PKIX path building failed ... unable to find valid certification path`，但同事用瀏覽器 / curl 打同一個 URL 冇問題。點排查？點修？

### 答法

- **意思**：JVM 用自己嘅 truststore（預設 `$JAVA_HOME/lib/security/cacerts`，唔係 OS 證書庫）驗唔到對方證書鏈去到一個信任嘅根 CA。
- **常見原因**：
  1. 對方**換咗證書**，但伺服器**冇送中間證書**——瀏覽器會自己補（AIA 下載 / 快取），Java 唔會 → 只有 Java 掛。
  2. 對方換咗**新 CA / 新根**，舊 JDK 嘅 `cacerts` 未有（JDK 太舊）。
  3. 公司代理 / 防火牆做 TLS 攔截，換咗佢自己簽嘅證書；或者應用指定咗自訂 truststore（`-Djavax.net.ssl.trustStore`）入面冇呢張根。
- **排查**：
  - `openssl s_client -connect host:443 -servername host -showcerts` 睇對方實際送咗幾張證書、鏈係咪完整、issuer 係邊個。
  - 喺**出事嗰台機、用同一個 JDK** 開 `-Djavax.net.debug=ssl:handshake`（或 `all`）睇握手同用緊邊個 truststore。
  - `keytool -list -cacerts | grep -i <CA名>` 確認根 CA 喺唔喺。
- **修法**：鏈唔完整 → 叫對方補中間證書（根治）；缺根 → 升級 JDK 或將**根 / 中間 CA**（唔係對方葉子證書）`keytool -importcert` 入應用嘅 truststore；**絕對唔好** trust-all / 關 hostname 校驗。
- **預防**：證書到期 / 換證監控；truststore 跟 JDK 升級；同合作方約定換證提前通知。

**面試一句話**：瀏覽器得、Java 唔得，九成係中間證書冇送或 JDK truststore 缺根——用 `s_client` 睇鏈、`javax.net.debug` 睇握手，補鏈或補 CA，唔好 trust-all。

### 追問 / 陷阱：TLS 1.2 vs 1.3 握手成本

- TLS 1.2 完整握手 **2-RTT**；TLS 1.3 **1-RTT**（ClientHello 就帶 key share），會話恢復可 **0-RTT**（但 0-RTT 數據可被**重放**，只適合冪等請求）。
- 後端真正嘅性能坑通常唔係版本，而係**每個請求都重新握手**：冇連接池 / keep-alive 被關 / 每次 new 一個 HttpClient。詳見 [HTTP keep-alive 與連接池](http-keepalive-connection-pool.md)。
