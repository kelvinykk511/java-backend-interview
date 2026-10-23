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

## 深入：揀型規則同排查（2026-10-23 warm-up）

**面試題**：Spring AOP 點樣揀 JDK 代理定 CGLIB？有咩分別同陷阱？

### 原理（容器入面係代理）

- Spring AOP 係**運行時代理**：Bean 初始化後由 `BeanPostProcessor`（`AbstractAutoProxyCreator.postProcessAfterInitialization`；循環依賴時經 `getEarlyBeanReference`）包一層代理再放入容器。
- 注入俾人嘅係**代理**，唔係原對象。

### JDK 動態代理

- 基於**介面**：`Proxy.newProxyInstance` + `InvocationHandler`；代理類實現同一介面，只能攔**介面方法**。
- 注入時要用**介面類型**；用實現類類型注入會 `BeanNotOfRequiredTypeException`（Boot 預設 CGLIB 就係為咗避開呢個坑）。

### CGLIB

- 生成目標類**子類**，覆寫方法 + `MethodInterceptor`；**唔需要介面**。
- `final` 類／`final` 方法／`private` 方法覆寫唔到 → **唔會被攔截**（靜默失效）；`static` 方法亦唔得。
- Spring 4+ 用 Objenesis **唔調構造器**；Spring 已 repackage CGLIB 入 `spring-core`。

### 默認規則

- **純 Spring Framework**：有介面 → JDK；冇介面 → CGLIB。
- **Spring Boot 2.0+**：`spring.aop.proxy-target-class=true` 預設 → **一律 CGLIB**（避免按實現類注入出錯）。

### 共同陷阱

- **自調用**（`this.foo()` 繞過代理，`@Transactional`／`@Async`／`@Cacheable` 失效）→ 拆 Bean、注入自己（`@Lazy`）、`AopContext.currentProxy()`（需 `spring.aop.expose-proxy=true`）。
- **CGLIB 代理對象嘅字段係空**：子類實例冇行構造器／冇注入，直接讀 `proxy.field` 拎到 `null`，要經 **getter**（會轉發到 target）。
- **`final` 方法上嘅 `@Transactional` 冇效**；Kotlin 類默認 `final` → 用 **kotlin-spring（all-open）** 插件。

### 性能 vs AspectJ

- JDK8 之後 JDK／CGLIB 差距好細，**唔係揀型依據**；揀型睇有冇介面、注入類型。
- **AspectJ**：編譯期／加載期織入（LTW），可攔自調用／`private`／字段／構造器；代價係構建或 agent 複雜。

### 追問：`@Configuration` 點解都係 CGLIB？

- `proxyBeanMethods=true`（預設）時 CGLIB 增強 `@Bean` 方法——**互相調用都返回容器單例**。
- 設 `proxyBeanMethods=false`（lite mode）可提速啟動，但 `@Bean` 互調會 **new 新對象**。

### 排查

- `AopUtils.isAopProxy`／`isJdkDynamicProxy`／`isCglibProxy`。
- `getClass()` 見 `$$SpringCGLIB$$`（Spring 6）或 `$Proxy`。
