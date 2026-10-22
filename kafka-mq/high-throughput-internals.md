# Kafka 點解咁快（高吞吐原理）

## 面試題
Kafka 用磁盤存消息，點解仲可以做到每秒幾十萬甚至百萬條？

## 答案要點
- **順序追加寫**：每個分區係 append-only 嘅 log segment，磁盤順序 IO 遠快過隨機 IO，接近內存速度。
- **靠 OS page cache**：Kafka 唔喺 JVM heap 自己做緩存，寫入落 page cache 就返回，由 OS 異步 flush；可靠性靠多副本（acks=all + min.insync），唔靠每條 fsync。好處：冇 GC 壓力、broker 重啟後緩存仍喺 OS 度。
- **零拷貝 sendfile**：consumer 拉數據時用 `FileChannel.transferTo`，數據由 page cache 直接去網卡，唔經用戶態，少咗兩次拷貝同上下文切換。
- **批量 + 壓縮**：producer 用 `batch.size` / `linger.ms` 攢一批先發；整個 RecordBatch 壓縮（lz4/zstd），broker 原樣存、consumer 先解壓，端到端都係批。
- **分區並行**：topic 拆多個分區，分散喺多台 broker，consumer group 按分區水平擴展。
- **稀疏索引**：`.index` / `.timeindex` 每隔一段先記一個 offset→位置，用 mmap + 二分查找定位。
- **網絡模型**：Reactor（Acceptor → Processor 線程 → RequestHandler 線程池），IO 同業務處理分開。

## 追問 / 陷阱
- **零拷貝一定生效？** 唔係。開咗 SSL/TLS 就要將數據讀入用戶態加密，sendfile 用唔到。
- **broker 會唔會重新壓縮？** 如果 topic 嘅 `compression.type` 同 producer 用嘅唔一樣（例如 producer lz4、topic 設 gzip），broker 要解壓再壓，CPU 飆、吞吐跌。保持 `compression.type=producer`。
- **分區越多越快？** 唔係。分區太多 → 同時寫好多 segment 檔，順序寫變近似隨機寫；檔案句柄多；leader 選舉/恢復更慢；producer 每分區一個 batch 攢得慢。

## 一句話
順序寫 + page cache + 零拷貝 + 批量壓縮 + 分區並行。
