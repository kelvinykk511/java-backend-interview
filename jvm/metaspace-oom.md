# Metaspace 與 OOM

## 面試問題

`OutOfMemoryError: Metaspace` 跟堆 OOM 差在哪？Metaspace 放什麼、常見成因、怎麼防？

## 答法

**Metaspace 放什麼（Java 8+）**

- 類別**元資料**：klass、方法元資料、常數池、註解等（以前在 PermGen）。
- 在**本地記憶體**（native），**不在堆**；預設可長到本機上限，可用 `-XX:MaxMetaspaceSize` 設硬頂。

**vs 堆**

| | Heap | Metaspace |
|---|---|---|
| 放什麼 | 實例對象 | 類別元資料 |
| OOM 典型 | 洩漏、無界快取、一次載太大 | 類別／ClassLoader 卸不掉、動態類爆量 |
| 調參直覺 | `-Xmx` | `-XX:MaxMetaspaceSize` |

**常見成因**

- 動態產生大量 class：熱部署、CGLIB／JDK Proxy 大量生成、Groovy／字節碼框架、每次請求 `new ClassLoader`。
- **ClassLoader 洩漏**：Loader 被靜態集合／ThreadLocal／緩存引用 → 對應類元資料**卸載不了** → Metaspace 單調爬升。
- 容器／插件反覆加載卸載失敗。

**排查與防**

- 看 loaded classes／classloader 數量是否持續上升（`jcmd GC.class_histogram`、`jmap -clstats`、或監控 NMT／Metaspace used）。
- 對比重啟前後；查熱部署頻率、是否每次請求新建 Loader。
- 根治：修洩漏、限制動態類產生；`MaxMetaspaceSize` 只是**保險絲**，調大只延後爆炸。

**面試一句話**：堆 OOM 看對象；Metaspace OOM 看**類別與 ClassLoader 是否卸不掉**。

## 常見追問 / 陷阱

「把 `MaxMetaspaceSize` 調大就好了？」

→ 只能延後。根因多半是 **ClassLoader／動態類洩漏**。調大前先確認 loaded classes 是否單調上升。另：`OutOfMemoryError: Direct buffer memory` 是堆外 NIO，**別跟 Metaspace 混談**。

## 小練習

線上 Metaspace used 緩升、loaded classes 也緩升，重啟後歸零再爬。你會先查熱部署／動態代理，還是先把 MaxMetaspaceSize 加倍？為什麼？
