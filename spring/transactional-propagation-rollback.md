# @Transactional 傳播行為與回滾

## 面試問題

`@Transactional` 默認傳播行為是什麼？什麼情況事務**不會**回滾？

## 答法

- 默認傳播：**`REQUIRED`** — 有事務就加入，沒有就新建。
- 默認只對 **RuntimeException / Error** 回滾；**checked exception**（如 `IOException`）默認不回滾，除非 `rollbackFor = Exception.class`。
- 方法必須經 **Spring 代理** 調用才生效：同類內部 `this.xxx()` 自調用通常**不開事務**。
- 常用搭配：`readOnly = true` 給純查詢；寫操作別亂開。

## 常見追問 / 陷阱

「`REQUIRED` 和 `REQUIRES_NEW` 差在哪？」

→ `REQUIRED` 加入外層同一事務，一起提交/回滾；`REQUIRES_NEW` 掛起外層、開新事務，內層提交了外層再回滾也回不掉內層已提交的。

## 小練習

一個 Service 方法 A 調同類私有方法 B（兩邊都標了 `@Transactional`），A 裡拋 RuntimeException，B 的寫入會回滾嗎？為什麼？
