# Spring 事務傳播行為

## 面試問題

`@Transactional` 的傳播行為裡，`REQUIRED`、`REQUIRES_NEW`、`NESTED` 差在哪？其餘幾個一句講場景。跟 Outbox 怎麼配才不會踩坑？

## 答法

（前提：經 **Spring 代理** 調用才生效；同類 `this.xxx()` 自調用通常不走代理。）

**三個必講清楚的**

| 傳播 | 行為（白話） | 適合場景 |
|------|--------------|----------|
| **REQUIRED**（預設） | 有就加入，沒有就新建；同一事務一起提交／回滾 | 大多數業務寫入 |
| **REQUIRES_NEW** | **掛起**外層，一定開**新事務**；內層先提交後，外層再回滾也回不掉內層 | 審計日誌、嘗試次數「一定要留下」、與主單成敗解耦 |
| **NESTED** | 同一**物理事務** + **savepoint**；內層回滾到 savepoint，外層可繼續 | 批次裡「某一筆失敗可跳過，整批還要提交」 |

對比口條：

- `REQUIRES_NEW`：真的兩筆 DB 事務；內層已提交對外層不可見回滾。
- `NESTED`：還在外層事務裡，靠 savepoint；**外層最終回滾，內層改動也沒了**。
- 別和 `@Async`／新線程搞混：異步預設不在原事務，要另開得自己標。

**其餘一句帶過**

| 傳播 | 場景一句 |
|------|----------|
| **SUPPORTS** | 有事務就加入，沒有就非事務跑（純查詢可搭 `readOnly`） |
| **NOT_SUPPORTED** | 掛起外層，非事務執行（避免長讀佔連線／鎖） |
| **NEVER** | 有事務就丟異常（強制「此 API 不准在事務裡」） |
| **MANDATORY** | 沒有外層事務就丟異常（強制呼叫端已開事務） |

**跟 Outbox**

- 寫業務表 + 寫 outbox 通常同一 **`REQUIRED`**：要麼一起成功，要麼一起回滾——這才是 outbox 保證「有業務變更就有待發事件」的前提。
- **別**對 outbox 單獨 `REQUIRES_NEW`、卻讓業務回滾：會留下「幽靈事件」或相反的不一致。需要「業務失敗也留痕」的是**審計日誌**，不是 outbox 主路徑。

## 常見追問 / 陷阱

「同類自調為什麼傳播失效？」

→ `this.method()` **繞過代理** → `@Transactional`（含傳播）不生效。解法：注入自身（或 `AopContext`）、拆到另一 Bean，或 AspectJ 編織。

「`REQUIRES_NEW` 開太多？」

→ 外層大事務裡反覆開小事務 → **連線占用**變高、中間出現**不一致窗口**（內層已見、外層未定）。審計／必要解耦再用，別當預設。

「下單失敗要回滾訂單，但嘗試次數日誌一定留下？」

→ 日誌方法用 **`REQUIRES_NEW`**（或獨立非事務寫）。若用預設 `REQUIRED`，主單回滾會把日誌一併回滾。

## 小練習

批次入帳 100 筆，第 37 筆校驗失敗要跳過並記錯，其餘要提交。選 `REQUIRES_NEW` 還是 `NESTED`？外層最後若整批回滾，兩種結果差在哪？Outbox 跟業務寫入該不該拆成 `REQUIRES_NEW`？

## 深入：catch 之後同 rollback-only（2026-11-23 warm-up）

**面試題**：外層 `REQUIRED` 調另一個 Bean 嘅 `REQUIRED` 方法，內層拋 `RuntimeException`，外層 catch 住當成功 return，會點？失敗審計用 `REQUIRES_NEW`、outbox 用 `REQUIRES_NEW`，分別啱唔啱？

- **同一條事務已被標 `rollback-only`**。內層 `REQUIRED` 加入外層事務，拋出 `RuntimeException`／`Error`（默認）就會標回滾。外層就算 catch 住、唔 rethrow，提交階段一樣會丟 **`UnexpectedRollbackException`**，整單回滾。呼叫方若只看自己「已經 handle」，會誤以為成功。
- **喺同一個 `REQUIRED` 裡寫失敗記錄**：記錄同主單一齊回滾，最後連痕跡都冇。
- **內層改 `REQUIRES_NEW`**（必須經代理、另一個 Bean 或 `AopContext`）：內層係獨立事務，回滾**唔會**標花外層。異常仍然會拋出；外層 catch 住就可以提交自己嗰段。呢個先係「成敗解耦」。代價係外層未結束時會**多佔一條 DB 連線**，同出現「內層已提交、外層未定」嘅窗口。
- **失敗審計／嘗試次數**用 `REQUIRES_NEW` 啱（主單回滾都要留痕）。**Outbox 唔可以**：事件同業務必須同一 `REQUIRED`，否則業務回滾仍會留下待打款事件。見 [Transactional Outbox](../distributed/transactional-outbox.md)。
- **自調用**：`this.logFailure()` 就算標咗 `REQUIRES_NEW` 都唔開新事務，行為同普通方法，記錄會跟外層一齊沒。

### 追問／陷阱

- 「catch 住就當提交成功」✗。`REQUIRED` 參與者拋 unchecked，外層再 succeed 都提交唔到。要跳過某段，用 `NESTED`（savepoint）或獨立 `REQUIRES_NEW`，而且要想清楚已提交部分回唔回得轉。
- 默認只對 **RuntimeException／Error** 回滾；checked exception 要自己寫 `rollbackFor`。
- 唔好喺事務方法裡面隨便 catch 住所有異常又唔 rethrow——要麼讓事務框架處理，要麼好清楚呢次係「仍然提交」定「一定回滾」。
