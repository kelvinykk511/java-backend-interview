# Spring Bean Scopes

## 面試問題

Spring Bean 預設 scope？`prototype` 每次拿都新建嗎？

## 答法

- 預設 **`singleton`**（一個 Spring 容器內一份）。
- **`prototype`**：每次 `getBean` 新建。
- 若被 singleton 注入，通常**只會拿到一次**（注入時創建）。要每次新實例可用 `ObjectFactory` / `@Lookup`。

## 常見追問

還有哪些 scope？→ `request` / `session` / `application`（Web 環境）。

---

## 深入（2026-11-05）：prototype 注入 singleton 失效 + request scope 陷阱

### 面試問題

`OrderService`（singleton）裏面 `@Autowired` 一個 `@Scope("prototype")` 嘅 `PriceCalculator`，以為每次調用都係新實例，結果多線程下計錯數。點解？點修？

### 答法

- **注入只發生一次**：singleton 創建時做屬性填充，嗰陣 `getBean(PriceCalculator)` 一次，之後一直用同一個對象 → prototype 實際變成「singleton 嘅私人單例」，有狀態就線程不安全。
- **修法（由推薦到唔推薦）**：
  1. **`ObjectProvider<PriceCalculator>`**：每次 `provider.getObject()` 先真正建新實例，最清晰、易測試。
  2. **`@Lookup` 方法**：Spring 用 CGLIB 覆寫方法每次 `getBean`；類同方法唔可以 `final`，`@Bean` 工廠方法建嘅 bean 用唔到。
  3. **Scoped proxy**：`@Scope(value="prototype", proxyMode=ScopedProxyMode.TARGET_CLASS)` → 注入嘅係代理，**每次調用方法都新建一個**（連續兩次 call 係兩個唔同對象，狀態會丟），通常唔係你想要。
  4. 直接 `applicationContext.getBean()`：得，但耦合容器。
- **最好嘅答案**：先問點解要有狀態——改成無狀態（狀態做參數 / 局部變量）就唔使 prototype。
- **prototype 生命週期**：Spring 建完就唔管，**`@PreDestroy` 唔會被調用**，持有資源要自己釋放。

### 追問 / 陷阱：request scope 喺異步線程

- `@RequestScope` bean 預設係 scoped proxy，背後靠 `RequestContextHolder`（**ThreadLocal**）攞當前請求。
- 喺 `@Async` / `CompletableFuture` / 線程池入面用 → `IllegalStateException: No thread-bound request found`。
- 修：喺請求線程先攞出需要嘅值當參數傳落去；或者用 `TaskDecorator` 複製上下文（記得 finally 清理，見 [ThreadLocal / TTL](../concurrency/threadlocal-ttl.md)）。唔好將 request 對象本身傳去異步線程——請求結束後佢可能已被回收重用。
- singleton 注入 request scope bean 如果**冇 proxy**，啟動就會報 `Scope 'request' is not active`。
