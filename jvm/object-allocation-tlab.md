# 對象分配流程：TLAB、大對象、晉升老年代

## 面試問題

`new` 一個對象，JVM 喺堆入面點分配？多線程同時 new 點解唔會搶同一塊內存？

## 答法

1. **類加載檢查** → 未加載就先加載（見 [類加載](classloader-parent-delegation.md)）。
2. **逃逸分析**：唔逃逸可能做標量替換，根本唔上堆（見 [JIT 逃逸分析](jit-escape-analysis.md)）。
3. **TLAB（Thread Local Allocation Buffer）**：每個線程喺 Eden 預先領一小塊，自己內部用**指針碰撞（bump the pointer）**分配，**唔使加鎖**——呢個就係多線程 new 唔打架嘅原因。TLAB 預設開（`-XX:+UseTLAB`），佔 Eden 好細比例。
4. TLAB 唔夠：剩餘空間細就丟棄、領新 TLAB；對象大過 TLAB 就喺 **Eden 共享區用 CAS** 分配。
5. Eden 都唔夠 → **Minor GC / Young GC**。
6. **大對象**：G1 入面超過 Region 一半 = **Humongous 對象**，直接放 Humongous Region（老年代性質）；Serial/ParNew 用 `-XX:PretenureSizeThreshold`。
7. 分配完：清零（所以字段有預設值）→ 設對象頭（見 [對象布局](object-layout-compressed-oops.md)）→ 執行 `<init>` 構造器。

### 晉升老年代幾時發生

- 年齡到 `MaxTenuringThreshold`（預設 15，因為 age 只得 4 bit）。
- **動態年齡判斷**：Survivor 入面某年齡及以下嘅對象總和超過 Survivor 一半（`TargetSurvivorRatio=50`）→ 嗰個年齡以上直接晉升。
- **Survivor 放唔落**：Minor GC 後存活對象太多 → 直接晉升（premature promotion），係老年代莫名其妙變大嘅常見原因。

## 追問 / 陷阱

- **「Humongous 分配頻繁」點睇**：GC log 見 `G1 Humongous Allocation` 觸發 GC。典型係一次性讀大 JSON / 大 `byte[]` / 大 List。修：**分批／流式處理，唔造大對象**。調大 `-XX:G1HeapRegionSize`（1M~32M，2 嘅冪）只對「半區以下」有效——**Region 最大 32MB，半區=16MB，所以 >16MB 永遠係 Humongous**，加大 Region **救唔到** 20MB `readAllBytes`；仲可能令 Young 區變粗、GC 停頓特性改變。有時會伴隨 `to-space exhausted`（連續 Region／疏散空間唔夠）。
- **TLAB 浪費**：線程好多（例如幾千個平台線程）每個都揸住一塊 TLAB，Eden 碎片化、Young GC 變頻。
- 指針碰撞要堆**規整**（有壓縮嘅收集器）；CMS 呢類標記-清除要用**空閒列表**。
