# 線上 Java 進程 CPU 100% 排查

**Q:** 線上某個 Java 服務 CPU 飆到 100%，你點樣喺幾分鐘內定位到係邊段代碼？

## 標準步驟
1. `top` 搵出高 CPU 的 Java 進程 PID。
2. `top -Hp <PID>` 搵出最食 CPU 的線程 TID。
3. `printf '%x\n' <TID>` 轉 16 進制（nid）。
4. `jstack <PID> | grep -A 30 'nid=0x<hex>'` 睇該線程堆棧，定位到具體方法。
5. 隔幾秒多抓 2–3 次 jstack，確認係同一位置（避免抓到瞬間）。
6. 工具：Arthas `thread -n 3` 一步到位；或 async-profiler 出火焰圖。

## 常見元兇
- 死循環 / 錯誤的自旋（CAS 無退避）、正則回溯爆炸。
- **GC 線程**食滿 CPU（線程名 `GC task thread` / `G1 Conc`）→ 其實係內存問題：`jstat -gcutil <PID> 1000` 睇 FGC 是否瘋狂增長，再 dump heap。
- 大量序列化 / 日誌拼字符串、HashMap 併發誤用（JDK7 擴容成環）。

## 追問 / 陷阱
- 「jstack 睇到係 GC 線程點算？」→ 唔好去改業務代碼，先查內存洩漏/大對象：`jmap -histo`、heap dump + MAT。
- 生產環境 `jmap -dump` 會 STW，大堆要喺流量摘除後做。
