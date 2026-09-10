# Gap Lock 與 Next-Key Lock

## 面試問題

InnoDB 在 RR 下怎麼避免幻讀？Gap Lock、Record Lock、Next-Key Lock 分別是什麼？

## 答法

三種鎖（簡化版）：

| 鎖 | 鎖什麼 |
|----|--------|
| Record Lock | 索引上記錄本身 |
| Gap Lock | 索引記錄之間的「空隙」（不含記錄） |
| Next-Key Lock | Record + 前面的 Gap（左開右閉區間） |

RR 預設用 **Next-Key** 鎖住範圍，擋住「別的事務往空隙插入」→ 抑制幻讀。

例子：`WHERE id = 5` 若 5 不存在，可能鎖住鄰近 gap；範圍條件 `id > 10 AND id < 20` 會鎖住區間內的 next-key。

實務注意：

- 等值查唯一索引且命中 → 常退化成 Record Lock（無 gap）
- Gap 會提高死鎖機率；寫衝突多可評估 RC + 業務防幻讀
- 沒合適索引 → 可能鎖更多（甚至近似表級），務必看 `EXPLAIN`

## 常見追問 / 陷阱

「RC 還有 Gap Lock 嗎？幻讀一定不會發生？」

→ RC 基本上不用 Gap Lock（主要 Record Lock）。幻讀在 RR 下「當前讀」（`SELECT … FOR UPDATE` / 更新）靠 next-key 擋住；快照讀看的是一致性視圖。別把「快照讀看不到新行」跟「當前讀加鎖防插入」混為一談。
