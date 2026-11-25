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

## 深入：幾分鐘定位同「其實係 GC」（2026-11-25 warm-up）

**面試題**：線上下單／行情服務 CPU 突然飆到 100%，你點喺幾分鐘內定位到邊段代碼？如果 `jstack` 見到嘅係 GC 線程，或者同事想喺交易 Pod 即刻 `jmap -dump`，你會點做？

- **先分清規模**。係一條線程打滿**單核**（死循環、熱路徑、正則回溯），定**所有核都滿**（GC、全站放量、錯誤自旋鋪開）。`top` 先睇進程，再 `top -Hp <pid>` 睇 TID。
- **對堆棧要可重複**。TID 轉 hex 做 `nid`，`jstack <pid>` 對應堆棧；隔幾秒再抓 2–3 次，確認唔係一瞬間。更快可以用 Arthas `thread -n 3`，或者 async-profiler 出火焰圖（看得清「係解釋執行、編譯碼、定 GC」）。
- **常見業務元兇**：無退避嘅 CAS 自旋、用戶輸入觸發嘅正則災難性回溯（`Pattern.matches` 遇上惡意 memo／備註）、瘋狂 JSON／日誌字串拼接、舊 JDK `HashMap` 併發擴容成環。
- **高 CPU 線程名係 `GC task thread` / `G1 Conc` / `VM Thread`**：先當**內存問題**，唔好先改業務熱路徑。`jstat -gcutil <pid> 1000` 睇老年代同 FGC 是否持續爬；Full GC 後回唔到穩定水位先考慮洩漏。再決定要唔要 heap dump，見 [heap OOM 排查](heap-oom-troubleshooting.md)。
- **生產 dump 要先摘流量**。`jmap -dump`／大堆 MAT 前會 STW；交易高峰喺線上 Pod 直接 dump，等於自己製造一次長停頓。先從 LB／摘除就緒，再 dump。加大 `-Xmx` 喺洩漏場景只係延遲爆炸，而且 Full GC 更長。
- **工具本身都會停應用**。`jstack` 都要入 safepoint；尖峰時 dump「好耐先出結果」可能係 Time-to-Safepoint 慢，唔等於工具壞咗。見 [Safepoint](safepoint.md)。

### 追問

- 「加 CPU limit／加 replica 就得？」如果係單線程死循環或 GC 抖動，加副本只係複製故障；要先定位堆棧同內存。
- 容器入面 `top` 見到 100% 要對照 **cgroup 配額**（一核限制下，一條線程就可以顯示 100%），同整機所有核打滿唔係同一件事。
