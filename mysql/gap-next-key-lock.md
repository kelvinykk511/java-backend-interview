# Gap Lock / Next-Key Lock

## 面試問題

InnoDB 在 RR 下怎麼防幻讀？Gap Lock、Record Lock、Next-Key Lock 差在哪？為什麼兩個事務往「同一個空隙」插**不同**主鍵仍可能死鎖？

## 答法

**三種鎖（記公式）**

| 鎖 | 鎖什麼 |
|----|--------|
| Record Lock | 索引上記錄本身 |
| Gap Lock | 索引記錄之間的**間隙**（不含記錄），阻止他事務 INSERT 進該區間 |
| Next-Key Lock | **Record + Gap**（常見記法：左開右閉區間） |

RR 下防幻讀的主力是 **Next-Key**：範圍／部分等值的**當前讀**會鎖住「已有行 + 行前空隙」，別的事務插不進 → 同一事務兩次當前讀不會突然「多出行」。

**幻讀場景（講清楚）**

- 幻讀指：同一事務兩次**當前讀**（`SELECT … FOR UPDATE` / `UPDATE` / `DELETE`）看到「多出來的行」。
- 普通 `SELECT` 在 RR 是快照讀（Read View），看不到他事務新提交的行——那是另一套機制；**別跟 next-key 混談**。
- 跟昨天 [RC vs RR](./rc-vs-rr.md) 連起來：RR = **事務級 Read View**（快照讀）+ **next-key／gap**（當前讀防插入）；當前讀一律走鎖，不是「RR 只靠快照」。

**RC vs RR（鎖）**

- **RC**：通常**沒有 gap lock**（可往空隙插）；當前讀主要 Record Lock → 插入死鎖少很多，但仍有行鎖死鎖。
- **RR**：range／部分等值當前讀可能加 **next-key／gap** → 防幻讀，也更容易 gap 相關死鎖。

**Insert Intention（死鎖經典題）**

- 插入前要先在目標 gap 上拿 **Insert Intention**；若該 gap 已被他事務的 gap／next-key 鎖住 → 等待。
- 多個事務要插**不同 row** 進**同一 gap**，卻各自先持有（或互相等待）該 gap 上的鎖 → 互相等 → **死鎖**。
- 口條：不是「主鍵撞車」才死鎖；**同一空隙上的意向鎖對撞**就夠。

**陷阱**

- **唯一索引等值且命中**：常退化成 **Record Lock**（不再鎖 gap）。
- **非唯一索引／範圍掃描／等值未命中**：更容易加 gap／next-key，鎖範圍也更大。
- 沒合適索引 → 鎖範圍膨脹（甚至接近表級）；`EXPLAIN` 看用哪條索引。
- 寫衝突多、可接受「同事務兩次讀可能不同」時，很多業務改 **RC + 業務防幻**，就是為了少 gap。

## 常見追問 / 陷阱

「RR 下對非唯一索引做 range 當前讀，兩個事務插入『不同』新主鍵為什麼還可能死鎖？」

→ 兩個 INSERT 落在**同一被鎖住的 gap** 裡；各自要 Insert Intention，又可能與對方已持有的 gap／next-key 形成等待環——跟新主鍵是否相同無關。

## 小練習

`SELECT … FOR UPDATE` 掃 `status=1`（非唯一二級索引），事務 A、B 同時再各插一筆 `status=1` 但主鍵不同。畫一下誰鎖了哪個 gap、Insert Intention 怎麼互等。


---

## 深入（2026-11-11）：唯一索引「唔存在」嘅 FOR UPDATE，同「先查再插」陷阱

### 面試問題

RR 下用「先 `SELECT … FOR UPDATE` 再決定是否 `INSERT`」做冪等（例如唯一鍵 `client_oid`）。若呢行**仲未存在**，其實鎖了什麼？兩個事務用**同一個**新 oid、同用**兩個不同**嘅新 oid，分別會點？改 RC 呢？

### 答法

**先溫三種鎖同「何時先有鎖」**

- Record／Gap／Next-Key 嘅定義見上文。普通 `SELECT` 喺 RR 係**快照讀**，唔加呢啲鎖；`FOR UPDATE`／`UPDATE`／`DELETE` 先係**當前讀**。
- **唯一索引等值且命中**：常退化成 **Record Lock**，唔鎖 gap。
- **唯一索引等值但唔命中**（行仲未在）：會鎖住「呢個鍵應該插入嘅 **gap**」（鄰近已有記錄之間的空隙，極端時空表／稀疏區間會去到 **supremum** 一類虛擬記錄，等於鎖住好大一段）。目的係擋住別人插入呢條「幻影行」。

**同一個新 oid**

- T1 `SELECT … FOR UPDATE` 唔中 → 持有該 gap；T2 用同一個 oid 再 `FOR UPDATE` 或 `INSERT`，要喺同一 gap 拿鎖／Insert Intention → **等 T1**。
- T1 再 `INSERT` 並提交後，T2 醒來可能見到行已在，或 `INSERT` 撞唯一鍵（Duplicate）。呢條路徑比較多係「串行 + 重複鍵」，唔一定死鎖；若兩邊仲持有其他鎖、順序相反，先至容易死鎖。

**兩個不同嘅新 oid（面試加分）**

- 若兩個鍵落喺**同一條 gap**（中間冇其他已有記錄），RR 下 T1 為咗「唔存在嘅 A」鎖咗成段 gap，T2 插 B 都要等——**主鍵／oid 根本唔撞車，都會互擋**。
- 空表、冷區、UUID 剛好相鄰，都可能把大量「唔相關」的插入打進同一 gap → 併發崩潰，甚至同 Insert Intention 形成**等待環死鎖**。
- 口條：Gap 鎖嘅粒度係**空隙**，唔係「只鎖你查嘅嗰個值」。

**RC 差在邊**

- RC 一般**唔持有 gap lock**（插入時仍有 Insert Intention／唯一約束檢查）。
- 兩個事務用不同 oid：多數唔會因「同一空隙」互相擋。
- 同一個新 oid：兩邊都可以見到「未存在」，再搶 `INSERT`，一個成功、一個 Duplicate——要用唯一索引兜底，而唔係靠 RR 的 gap 做業務互斥。
- 改 RC **減 gap 死鎖**，但唔等於冇行鎖死鎖，亦唔再靠 next-key 擋當前讀幻讀。

**業務建議（CEX 下單／充值 txid 好常見）**

- 冪等優先：**唯一索引 + 直接 INSERT**，撞鍵當「已受理」，唔好用「先 FOR UPDATE 再插」當並發鎖。
- 真要鎖已有行，先確保等值命中唯一索引；範圍／非唯一二級索引嘅當前讀，鎖範圍可以好大。
- 讀 `SHOW ENGINE INNODB STATUS`／`performance_schema.data_locks` 時，留意 `GAP`／`INSERT INTENTION`／`supremum`，唔好只揾 record 對撞。

### 追問／陷阱

- 「唯一索引就一定冇 gap？」→ 只係**等值命中**時常退化；**唔命中**同範圍掃描仍然有 gap／next-key。
- 「兩個 INSERT 主鍵都唔同，點會死鎖？」→ 同一 gap 上的 gap／Insert Intention 對撞就夠，見上文 Insert Intention。
- 「改 RC 之後仲有死鎖就怪了？」→ 行鎖順序、唯一鍵衝突等待，RC 一樣可以死鎖；只係少咗 gap 呢一類。
