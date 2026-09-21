# Gap Lock 與 Next-Key Lock

## 面試問題

InnoDB 在 RR 下怎麼避免幻讀？Gap Lock、Record Lock、Next-Key Lock 分別是什麼？什麼情況下容易因 Gap 死鎖？

## 答法

三種鎖（簡化版）：

| 鎖 | 鎖什麼 |
|----|--------|
| Record Lock | 索引上記錄本身 |
| Gap Lock | 索引記錄之間的「空隙」（不含記錄） |
| Next-Key Lock | Record + 前面的 Gap（左開右閉區間） |

RR 預設用 **Next-Key** 鎖住範圍，擋住「別的事務往空隙插入」→ 抑制幻讀。

例子：`WHERE id = 5` 若 5 不存在，可能鎖住鄰近 gap；範圍條件 `id > 10 AND id < 20` 會鎖住區間內的 next-key。

**容易死鎖的場景（面試加分）**

- 兩事務對同一範圍做 `SELECT … FOR UPDATE` / 範圍更新，再各自往 gap 插入 → Gap 互等。
- 唯一鍵衝突：A 插入失敗留 gap 意向，B 再插同一鍵，再配合其他鎖順序 → 經典「插入意向鎖」死鎖。
- 沒合適索引 → 鎖範圍膨脹（甚至接近表級），`EXPLAIN` 必看。

實務注意：

- 等值查**唯一索引且命中** → 常退化成 Record Lock（無 gap）
- Gap 會提高死鎖機率；寫衝突多可評估 **RC + 業務防幻讀**
- 當前讀（`FOR UPDATE` / 更新刪除）才加這些鎖；普通 `SELECT` 在 RR 是快照讀

## 常見追問 / 陷阱

「RC 還有 Gap Lock 嗎？幻讀一定不會發生？」

→ RC 基本上不用 Gap Lock（主要 Record Lock）。幻讀在 RR 下「當前讀」靠 next-key 擋住；快照讀看的是一致性視圖。別把「快照讀看不到新行」跟「當前讀加鎖防插入」混為一談。
