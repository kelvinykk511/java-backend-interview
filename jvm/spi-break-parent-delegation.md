# SPI 與打破雙親委派

## 面試問題

為什麼 JDBC、`ServiceLoader` 這類 SPI 要「打破」雙親委派？`ThreadContextClassLoader` 在解什麼問題？

## 答法

- **雙親委派**：子 loader 先請父加載；核心類由 Bootstrap／Platform 載，避免 classpath 偽造 `java.lang.*`。
- **SPI 矛盾**：JDK 裡的 SPI 介面（如 `Driver`、`ServiceLoader`）由**啟動／平台類加載器**載入，但實作類（MySQL driver、業務插件）在**應用 classpath**。父 loader **看不到**子 classpath，若嚴格委派，介面載得到、實作載不到。
- **打破方式（面試講清楚一種即可）**：
  1. **SPI／ServiceLoader**：介面那邊用**執行緒上下文類加載器（TCCL）**去載實作（常見是 AppClassLoader）。
  2. **容器／熱部署**（Tomcat 等）：Webapp ClassLoader **先找自己**再委派，隔離各應用依賴。
- **類唯一性**仍是「全限定名 + ClassLoader」；兩邊各載一份同名類 → 易 `ClassCastException`。

## 常見追問 / 陷阱

「自己 `new` 一個 ClassLoader 載 plugin，為什麼 `MyService` cast 失敗？」

→ 介面若也被 plugin loader 再載一份，和主應用的 `MyService` 不是同一個類。介面應由**共同的父 loader**載，只讓實作走子 loader。
