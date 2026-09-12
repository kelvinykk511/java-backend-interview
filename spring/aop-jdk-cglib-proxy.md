# Spring AOP：JDK 動態代理 vs CGLIB

## 面試問題

Spring AOP 預設用哪種代理？什麼情況下會換成 CGLIB？自呼叫（`this.xxx()`）為什麼切不到？

## 模範答案（面試可講）

- Spring AOP 是**執行時代理**，不是改 bytecode 的 AspectJ 編譯期織入（預設）。
- **有介面**時：預設用 **JDK 動態代理**（`Proxy.newProxyInstance`），代理物件實作同一介面，呼叫走 `InvocationHandler`。
- **沒有介面**（或強制）時：用 **CGLIB** 產生目標類的**子類**，覆寫方法再插入通知。
- Spring Boot 2.x+ 常預設偏向 CGLIB（`spring.aop.proxy-target-class=true`），就算有介面也可能用 CGLIB。
- **自呼叫**：同一個類裡 `this.foo()` **不經過代理物件**，所以 `@Transactional` / `@Cacheable` 等 AOP 通知**不會生效**。解法：注入自身代理、拆 Bean、或用 AspectJ。
- 限制：`final` 類／方法 CGLIB 無法代理；`private`／`static` 方法通常切不到。

## 常見追問／陷阱

- 「介面裡的 default method、或呼叫未覆寫的父類方法」行為要小心；以及 **同一 Bean 內方法互調** 是最常考的陷阱。
- 別把「有 `@Transactional` 就一定開事務」講死——代理沒套上就等於沒有。

## 關聯

- [Bean 生命週期](bean-lifecycle.md)（何時建立代理）
- [Transaction 傳播與回滾](transactional-propagation-rollback.md)
