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
