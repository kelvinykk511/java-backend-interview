# Metaspace 與 OOM（面試）

## 面試問題

`OutOfMemoryError: Metaspace` 跟堆 OOM 差在哪？線上怎麼查、怎麼防？

## 答法

- **堆（Heap）**：放實例對象；OOM 常見是洩漏、快取無上限、一次載入太大。
- **Metaspace（Java 8+）**：放 **類別元資料**（以前在 PermGen）。本地記憶體，預設可長到本機上限（可用 `-XX:MaxMetaspaceSize` 卡住）。
- 典型成因：
  - 動態產生很多 class（熱部署、大量代理、Groovy/反射產生、錯誤的 ClassLoader 重複載入）
  - ClassLoader 洩漏：卸載不掉 → 元資料越積越多
- 排查方向：`jcmd`/`jmap` 看 classloader 與 loaded class 數量是否持續爬升；對比重啟前後；查是否反覆熱部署或每次請求 new ClassLoader。
- 面試一句話：堆 OOM 看對象；Metaspace OOM 看 **類別與 ClassLoader 是否卸不掉**。

## 常見追問 / 陷阱

「把 MaxMetaspaceSize 調大就好了？」

→ 只能延後爆炸。根因多半是 **ClassLoader / 動態類別洩漏**。調大前先確認 loaded classes 是否單調上升。另：`Direct buffer` OOM（`OutOfMemoryError: Direct buffer memory`）又是另一塊（堆外 NIO），別跟 Metaspace 混談。
