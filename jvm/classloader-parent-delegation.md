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
