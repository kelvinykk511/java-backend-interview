# 類加載與雙親委派

## 面試問題

Java 類加載是什麼？雙親委派模型在解決什麼問題？

## 答法

**誰加載**

- Bootstrap：核心 JDK（`rt.jar` 等）
- Extension / Platform：擴展庫
- Application：classpath 上的業務類
- 還可自訂 ClassLoader（熱部署、隔離插件）

**雙親委派（簡化）**

加載一個類時，先請父加載器試；父找不到，自己才加。好處：

1. **避免重複加載** 同一份類被多個 loader 各載一份
2. **核心類不被竄改** 業務 classpath 裡即使有同名 `java.lang.String`，也會先由 Bootstrap 載到真貨

類的唯一性 ≈ **全限定名 + 加載它的 ClassLoader**。兩個 loader 各載同一 `.class`，在 JVM 裡是兩個不同的類。

## 常見追問 / 陷阱

「為什麼要打破雙親委派？」

→ 例如 SPI、Tomcat 隔離 webapp、熱更新：子 loader 要先載業務/插件類，或故意不讓父看到。打破要說清楚隔離邊界，否則容易 `ClassCastException` / 鏈上兩份依賴打架。

姊妹篇：[SPI 與打破雙親委派](spi-break-parent-delegation.md)、[Metaspace OOM](metaspace-oom.md)

---

## 深入（2026-11-18）：點解會 CCE、點樣正確打破、同 Metaspace 洩漏

### 面試可以點追

業務 jar 入面放一個 `java.lang.String`，點解取代唔到 JDK 嗰個？兩個 ClassLoader 載「同一個類」點解會 `ClassCastException`？SPI／Tomcat／熱更新規則包要點打破先唔爆？

### 答法

**委派鏈（由上到下）**

- Bootstrap：核心 JDK 類（`java.lang.*` 等）。
- Platform（舊稱 Extension）：平台擴展。
- Application（系統／AppClassLoader）：應用 classpath。
- 之後可以再掛自訂 loader（插件、熱部署、Tomcat Webapp）。

**雙親委派做兩件事**

- 加載時**先請父、父唔得先自己加**。所以 classpath 即使有一份假 `java.lang.String`，都會由 Bootstrap 載到真貨，核心類唔被竄改。
- 同一個類盡量只被一個 loader 載一次，避免各寫一份各用各嘅。

**類嘅唯一性 = 全限定名 + 載佢嘅 ClassLoader**

- 兩個 loader 各 `defineClass` 同一份 `.class`，喺 JVM 係**兩個唔同嘅類**。
- 互相賦值／cast 就會 `ClassCastException`，訊息仲好迷惑：`com.foo.RiskRule cannot be cast to com.foo.RiskRule`。唔係程式寫錯包名，係**兩邊唔係同一個類對象**。

**咩情況要打破（講清一種就得）**

- **SPI**（JDBC `Driver`、`ServiceLoader`）：介面喺 JDK／父 loader，實作喺應用 classpath；父**睇唔到**子嘅類。常見做法係介面嗰邊用**執行緒上下文類加載器（TCCL）**去載實作。詳見 [SPI 與打破雙親委派](spi-break-parent-delegation.md)。
- **Tomcat／多應用隔離**：Webapp ClassLoader **先搵自己再委派**，避免兩個 war 嘅同名依賴互相污染。
- **熱更新規則／插件**：子 loader 載實作，方便卸掉舊版本。

**正確打破嘅邊界**

- **共享介面／API 由共同嘅父 loader 載**，只讓實作走子 loader。
- 插件 jar **唔好再打包一份**同名介面（shade 重複會直接 CCE）。
- 用完要斷引用（靜態集合、緩存、ThreadLocal），舊 loader 先有機會被回收。

### 追問／陷阱

- **每分鐘 `new URLClassLoader` 載規則 jar**：若舊 loader 被靜態 Map、線程池 ThreadLocal、連接、日誌 appender 引用住，類同元資料**卸唔掉**，`loaded classes`／Metaspace **單調上升**，最後 `OutOfMemoryError: Metaspace`。調大 `-XX:MaxMetaspaceSize` 只係延後；根治係洩漏同動態類數量。唔好同堆 OOM、`Direct buffer memory` 混為一談。見 [Metaspace OOM](metaspace-oom.md)。
- **`Thread.currentThread().setContextClassLoader` 之後唔還原**：線程池會把錯誤 TCCL 傳到下一個任務，SPI／框架可能載錯實作，或者令 loader 掛喺池線程上卸唔掉。
- 面試講「打破雙親委派」要同時講**隔離邊界**；只講「Tomcat 先自己再父」但唔講介面要共用，仍然會 CCE。
