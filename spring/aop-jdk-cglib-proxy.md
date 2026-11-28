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

## 深入：切面順序、鎖同事務、@Async 疊加（2026-11-28 warm-up）

**面試題**：同一個方法或一條呼叫鏈上同時有自定義分布式鎖切面、`@Transactional`、`@Async`，代理同執行順序會出咩問題？仲有邊啲情況注解會靜默失效？

### 先分清「攔唔到」同「攔到但順序錯」

- Spring AOP（proxy mode）只攔**經代理嘅外部呼叫**。以下會**靜默唔生效**（唔報錯）：
  - 同類 `this.foo()`（最常見）；
  - `@PostConstruct`／初始化階段就調業務方法（代理未必已就緒）；
  - **非 public** 方法上嘅 `@Transactional`（官方：proxy mode 只處理 public）；
  - `final` 類／方法（CGLIB 覆寫唔到）；Kotlin 類默認 `final`，要 all-open。
- 注入到嘅係代理，唔係 target。排查用 `AopUtils.isAopProxy`，以及 `getClass()` 係 `$$SpringCGLIB$$` 定 `$Proxy`。

### 切面順序

- 呼叫進入時，**外層切面先執行、後結束**（像洋蔥）。`@Order`／`Ordered` **數字愈細愈靠外**。
- `@Transactional` 預設 `Ordered.LOWEST_PRECEDENCE`，貼近業務方法（較內）。
- **分布式鎖應該在外、事務在內**：先攞鎖 → 開事務 → 業務 → commit → 放鎖。其他線程喺 commit 前攞唔到鎖，唔會按舊快照去寫。
- **相反（事務在外、鎖在內）**：等鎖期間佔住 DB 連線；放鎖之後事務可能仲未 commit，另一條線程攞到鎖會讀到舊數據再寫 → 超賣／重複扣。鎖粒度同事務邊界要對齊。

### `@Async` + `@Transactional`

- `AsyncAnnotationBeanPostProcessor` 預設 **`beforeExistingAdvisors=true`**，把 async advisor **插到現有鏈最前面**（最外）：先交去線程池，**喺 worker 線程**先至開事務。呢個先後通常先係啱嘅。
- 唔好自行把事務切面調到比 Async **更外**：否則呼叫線程開咗事務，`@Async` 立即返回，外層事務提早 commit；真正工作喺池入面跑，`TransactionSynchronizationManager` 呢類 ThreadLocal **帶唔走**。
- 實務：**唔好同一個方法同時標 `@Async` + `@Transactional`**。拆 Bean：請求線程走同步事務方法；發 MQ／通知用 `TransactionSynchronization.afterCommit`，或者另一個 Bean 嘅 `@Async`。
- `AopContext.currentProxy()`（要 `spring.aop.expose-proxy=true`）綁喺當前線程，**換線程之後唔能再用**。

### 追問／陷阱

- 以為加咗 `@Transactional`／`@DistributedLock` 就一定生效——自調用、非 public、`final` 都會靜默失效。
- 鎖同事務 `@Order` 寫反，高併發先爆。
- `@Async` 方法 `void` 嘅例外預設只入 `AsyncUncaughtExceptionHandler`，呼叫方捉唔到，唔好當成事務回滾已通知到上游。
