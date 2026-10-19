# Spring @Transactional 失效場景

## 面試問題

「`@Transactional` 有咩情況會唔生效或者唔 rollback？」

## 根本原因

Spring 事務靠 **AOP 代理** + 綁定喺 **ThreadLocal** 嘅 connection。繞過代理、換咗線程、或者例外冇傳到代理，都會失效。

## 答法（按「代理 → 異常 → 配置」三類講）

### A. 冇經過代理（事務根本冇開）
1. **同類自調用**：`this.methodB()` 唔經 Spring 代理 → B 上嘅 `@Transactional` 無效。解法：抽到另一個 Bean；注入自己（`@Lazy self`）；或 `AopContext.currentProxy()`（要 `exposeProxy=true`）。
2. **方法可見性 / 修飾**：`private` 一定失效；Spring 6.0 之前非 public 都唔生效（6.0 起 class-based 代理支援 protected / package-private）；`final` / `static` 方法 CGLIB 覆寫唔到。
3. **個類唔係 Spring Bean**：自己 `new` 出嚟嘅對象冇代理。

### B. 異常冇傳到代理（代理以為成功）
4. **異常俾自己 `try-catch` 食咗**，冇再拋出 → 照 commit。要 rethrow，或 `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`。
5. **拋 checked exception**（例如 `IOException`）：默認只對 `RuntimeException` / `Error` 回滾 → 要 `rollbackFor = Exception.class`。
6. **多線程 / `@Async`**：新線程拎唔到原本 ThreadLocal 嘅 connection，子線程嘅 SQL 唔喺同一個事務；子線程出錯父事務唔會 rollback。

### C. 配置或傳播設錯
7. **傳播行為設錯**：`NOT_SUPPORTED` / `NEVER` / `SUPPORTS`（冇外層時）根本冇事務；`REQUIRES_NEW` 開新事務，外層 rollback 唔會帶埋佢。
8. **多數據源用錯 TransactionManager**：事務開喺 A 庫，SQL 打去 B 庫；或者冇配置事務管理器。
9. **存儲引擎唔支持事務**（MyISAM）。

## 一句總結

> 繞過代理、例外冇傳出去、換咗線程，三類原因涵蓋大部分失效。

## 常見追問 / 陷阱

- **內層 `REQUIRED` 拋 RuntimeException，外層 catch 住繼續行**：內層已經將共享事務標記 rollback-only，外層 commit 時拋 **`UnexpectedRollbackException`**（*Transaction silently rolled back because it has been marked as rollback-only*）。想內層失敗唔影響外層，用 **`REQUIRES_NEW`** 或 **`NESTED`**（savepoint）。
- **長事務**：`@Transactional` 入面做 RPC / HTTP 調用，鎖同 DB connection 揸住好耐，拖垮連接池。遠程調用移出事務。
- **事務內發 MQ**：消息喺 commit 之前發出 → 下游可能查唔到資料（未 commit），或者事務 rollback 咗但消息已發。用 `@TransactionalEventListener(phase = AFTER_COMMIT)` / `TransactionSynchronization.afterCommit` 喺 commit 後發；要唔丟就用 [Transactional Outbox](../distributed/transactional-outbox.md)。

相關：[transactional-propagation-rollback](transactional-propagation-rollback.md)、[tx-propagation-scenarios](tx-propagation-scenarios.md)、[aop-jdk-cglib-proxy](aop-jdk-cglib-proxy.md)
