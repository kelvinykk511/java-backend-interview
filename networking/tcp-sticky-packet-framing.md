# TCP 粘包 / 拆包 + Netty 解碼器

> 面試問法：「TCP 會唔會粘包？點解？Netty 點處理？」

## 一句講晒
TCP 係**字節流**，冇「一條消息」嘅邊界；所以應用層自己要定協議切消息。「粘包 / 拆包」其實係**冇做消息分幀**嘅應用層問題，唔係 TCP bug。

## 點解會出現
- 發送端：Nagle 算法 / 寫緩衝將幾次細 write 合成一個 segment → 接收端一次讀到兩條（粘）
- 一條大消息超過 MSS / 接收緩衝 → 分幾次讀到（拆）
- 接收端 `read()` 一次讀幾多，同對方 `write()` 幾次**冇對應關係**
- UDP 冇呢個問題：一個 datagram 就係一條消息（但會丟、會亂序）

## 三種分幀方法
| 方法 | 例子 | Netty 解碼器 |
|---|---|---|
| 定長 | 每條 64 bytes | `FixedLengthFrameDecoder` |
| 分隔符 | `\r\n`（Redis RESP、HTTP header、FIX 用 SOH 分字段 + 長度） | `LineBasedFrameDecoder` / `DelimiterBasedFrameDecoder` |
| **長度字段 + 內容**（最常用） | 4 bytes 長度 + body；gRPC/HTTP2 frame | `LengthFieldBasedFrameDecoder` + 發送端 `LengthFieldPrepender` |

## Netty 點做
- `ByteToMessageDecoder` 內部有**累積緩衝**（cumulation）：字節唔夠一條就等，夠就切出一條交俾下一個 handler，剩低嘅留低等下次。
- `LengthFieldBasedFrameDecoder(maxFrameLength, lengthFieldOffset, lengthFieldLength, lengthAdjustment, initialBytesToStrip)`：記住 `lengthAdjustment` 係「長度字段之後到 body 嘅修正值」，最易填錯。

## 常見追問 / 陷阱
- **一定要設 `maxFrameLength`**：否則對方報個超大長度，decoder 會一直累積 → 內存（直接內存）爆。
- 解碼器**唔可以 `@Sharable`**（有每條連接自己嘅累積狀態），每個 channel 要 new 一個。
- 分隔符法：body 入面出現分隔符就要轉義；二進制數據用長度字段更穩。
- `TCP_NODELAY`（關 Nagle）只係減延遲，**唔會解決**粘包，仲係要分幀。
- WebSocket / HTTP 已經幫你分好幀；自定 TCP 協議（交易所行情 / 撮合網關）先要自己做。

## 相關
- [Netty 線程模型](netty-thread-model.md)、[NIO / Netty EventLoop](nio-netty-event-loop.md)、[TCP 三次握手](tcp-three-way-handshake.md)
