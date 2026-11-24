# Java 21 虛擬線程（Virtual Threads）

## 面試題
虛擬線程係咩？同平台線程／線程池有咩分別？咩場景啱、咩場景唔啱？

## 標準答案
- **平台線程**：1:1 對應 OS 線程，棧預留約 1MB，開幾千條就到頂；**虛擬線程**：JVM 調度，M:N 掛喺少量 carrier 線程（預設一個 ForkJoinPool，大小≈CPU 核數），棧幀放 heap、按需增長，可以開到幾十萬、百萬條。
- **核心機制**：遇到阻塞 IO（JDBC、HTTP、`sleep`、`BlockingQueue.take`）時 **unmount**——把棧存去 heap，carrier 去跑第二條虛擬線程；IO 返嚟再 mount。所以寫同步代碼，都攞到接近異步嘅吞吐。
- **提升嘅係吞吐，唔係單個請求延遲**：Little's law，併發數唔再被線程數卡住。CPU 密集任務冇幫助（核數就係上限）。
- **用法**：`Executors.newVirtualThreadPerTaskExecutor()`、`Thread.ofVirtual().start(...)`；Spring Boot 3.2+ `spring.threads.virtual.enabled=true`（Tomcat、`@Async` 等用虛擬線程）。
- **唔好池化虛擬線程**：佢平到用完即棄。要限流就用 `Semaphore` 限住下游（DB 連接池得 50 條，百萬虛擬線程只會全部排隊等連接，或者打爆下游服務）。

## 常見追問／陷阱
- **Pinning（釘住 carrier）**：JDK 21 喺 `synchronized` 區塊入面阻塞、或者行緊 native 方法時，虛擬線程唔能 unmount，carrier 被釘死；carrier 全部被釘 → 吞吐跌到得返「核數條線程」，甚至卡死。
  - 排查：JDK 21 用 `-Djdk.tracePinnedThreads=full` 或 JFR 事件 `jdk.VirtualThreadPinned`。
  - 解：熱路徑上嘅 `synchronized` + IO 改用 `ReentrantLock`；升級驅動（例如新版 MySQL Connector/J 已改用 `ReentrantLock`）。JDK 24（JEP 491）已解決 `synchronized` 造成嘅 pinning，但 native 調用仍會釘。
- **ThreadLocal**：一請求一虛擬線程，用 ThreadLocal 做「每線程緩存」（例如緩存 `SimpleDateFormat`、大 buffer）會失效兼食內存；傳上下文可考慮 Scoped Values（JDK 21 preview）。
- **唔係萬能**：瓶頸喺 DB／下游時，開虛擬線程只係將排隊由線程池搬去連接池。

## 一句話總結
虛擬線程 = 阻塞時讓出 carrier 嘅平價線程；IO 密集一請求一線程好用，記住唔池化、限住下游、避開 pinning。


## 深入：開咗虛擬線程但 QPS 冇升（2026-11-24 warm-up）

**面試題**：Spring Boot 開咗 `spring.threads.virtual.enabled=true`，下單 QPS 幾乎冇升，反而大量 HikariCP `Connection is not available`，JFR 見到 `jdk.VirtualThreadPinned`。虛擬線程係咪「開咗就加快」？邊啲寫法會令情況更差？

- **提升嘅係吞吐，前提係阻塞真係可以 unmount**。請求改用虛擬線程之後，若瓶頸喺 DB 連接池、下游限流、或 carrier 被釘死，排隊只係由 Tomcat 線程池搬去 Hikari／下游，QPS 唔會升，超時會更早爆。
- **對齊下游容量**。Hikari 仍然只有幾十條連接；百萬虛擬線程一齊 `getConnection`，就係連接池超時。要用 `Semaphore`（或等價限流）把併發對齊連接池／下游配額，而唔係再加一個虛擬線程池。
- **JDK 21 pinning**：熱路徑若係 `synchronized` 包住阻塞 IO（JDBC、Jedis/`RedisTemplate`、HTTP），虛擬線程唔能 unmount，**carrier 被釘住**。carrier 數量≈核數，全部被釘就等同一下退回「核數條平台線程」。查 JFR `jdk.VirtualThreadPinned` 或 `-Djdk.tracePinnedThreads=full`；熱路徑改 `ReentrantLock`，並升級已改鎖類型嘅驅動。JDK 24（JEP 491）修咗 `synchronized` pinning，**native／JNI 仍然會釘**。
- **唔好池化虛擬線程**。`ThreadPoolExecutor` 包住 VT「限制 core=200」會令排隊同掛起語義變複雜，失去「一任務一線程、阻塞即讓出」嘅好處。要限流就限**下游**，唔限虛擬線程本身。
- **ThreadLocal 陷阱**：一請求一虛擬線程，用 ThreadLocal 緩存 1MB buffer／`SimpleDateFormat` 會令堆按併發線性膨脹，甚至 OOM。短生命周期嘅虛擬線程唔適合做「每線程大緩存」；上下文可考慮 Scoped Values。
- **仲有唔適用**：CPU 密集（carrier 就係 ForkJoinPool，會互相搶核）、長時間持有連接／鎖先先做遠程呼叫。parallel stream 預設仍走 `ForkJoinPool.commonPool`，唔會自動改用虛擬線程。

### 追問

- 開咗虛擬線程之後 `synchronized` 工具類 + Redis 同步呼叫，點解比平台線程池更易卡死？（平台線程池最多釘住池內嗰批；VT 模式下少量 carrier 被釘，整個進程吞吐都會塌。）
- Spring Boot 3.2+ 開 `spring.threads.virtual.enabled` 只換 Tomcat／指定執行器；自己 `new Thread()`、自建池、訂閱 MQ 嘅 listener 容器要單獨改，唔好假設成個進程都切咗。
