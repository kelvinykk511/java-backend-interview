# Spring @Transactional 失效場景

## 面試問題

「你喺方法上面加咗 `@Transactional`，但出錯之後數據冇回滾。可能有咩原因？」

## 答法（按「代理 → 異常 → 配置」三類講）

### A. 冇經過代理（事務根本冇開）
1. **同類自調用**：`this.methodB()` 唔經 Spring 代理 → B 上嘅 `@Transactional` 無效。
   解法：拆去另一個 Bean；或注入自己 / `AopContext.currentProxy()`（要 `exposeProxy=true`）。
2. **方法唔係 public**（Spring 6 之前 protected/private 都唔生效；private 任何版本都唔得）。
3. **`final` / `static` 方法**：CGLIB 改寫唔到。
4. **個類唔係 Spring Bean**：自己 `new` 出嚟嘅對象冇代理。

### B. 異常冇傳到代理（代理以為成功）
5. **異常俾自己 `try-catch` 食咗**，冇再拋出 → 照 commit。要 rethrow，或 `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`。
6. **拋 checked exception**（例如 `IOException`）：默認只對 `RuntimeException` / `Error` 回滾 → 要 `rollbackFor = Exception.class`。
7. **另開線程 / `@Async` 做 DB 操作**：事務綁喺 ThreadLocal，新線程唔喺同一個事務入面，子線程出錯主事務唔會回滾。

### C. 配置或傳播設錯
8. **傳播行為**：`REQUIRES_NEW` 內層已獨立提交；`NOT_SUPPORTED` / `SUPPORTS`（外層冇事務）就冇事務。
9. **多數據源用錯 TransactionManager**：事務開喺 A 庫，SQL 打去 B 庫。
10. **存儲引擎唔支持事務**（MyISAM）。

## 一句總結

> 先問「有冇經代理」（自調用、非 public、final、非 Bean），再問「異常有冇拋到代理」（被 catch、checked、異步線程），最後睇「配置」（傳播、事務管理器、引擎）。

## 常見追問 / 陷阱

- 「內層 `REQUIRED` 拋異常、外層 catch 咗，會點？」→ 內層已標記 `rollback-only`，外層提交時拋 **`UnexpectedRollbackException`**，成個事務都回滾。
- 「事務入面發 MQ / 調外部接口？」→ 事務回滾咗但消息已發出；用 `@TransactionalEventListener(phase = AFTER_COMMIT)` 或 Outbox。
- 大事務：事務內做遠程調用會長時間佔住連接同鎖，生產上要縮短事務範圍。

相關：[transactional-propagation-rollback](transactional-propagation-rollback.md)、[tx-propagation-scenarios](tx-propagation-scenarios.md)、[aop-jdk-cglib-proxy](aop-jdk-cglib-proxy.md)
