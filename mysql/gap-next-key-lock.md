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
