# Spring Bean 生命週期

## 面試問題

Spring 容器裡一個 Bean 從創建到銷毀，大致經過哪些關鍵階段？`@PostConstruct` 和 `InitializingBean` 誰先誰後？

## 答法

- **大致順序**：實例化 → 屬性注入（依賴注入）→ Aware 回調（如 `BeanNameAware` / `ApplicationContextAware`）→ `BeanPostProcessor.postProcessBeforeInitialization` → 初始化（`@PostConstruct` → `InitializingBean.afterPropertiesSet` → 自訂 `init-method`）→ `BeanPostProcessor.postProcessAfterInitialization`（AOP 代理常在這裡包一層）→ 使用中 → 銷毀（`@PreDestroy` → `DisposableBean.destroy` → 自訂 `destroy-method`）。
- **記住兩點就夠面試**：先注入再初始化；銷毀時先註解/接口再開自訂方法。
- **實務**：業務初始化優先用 `@PostConstruct`（清楚、不綁 Spring 接口）；需要改所有 Bean 行為才寫 `BeanPostProcessor`。

## 常見追問 / 陷阱

「為什麼 `@Transactional` 在同類自調會失效？」

→ 代理通常在 **初始化之後** 才套上。同類內部 `this.method()` 不走代理，事務攔截器進不去（跟生命週期/代理時機有關）。

---

## 進階（2026-10-30）：BeanFactoryPostProcessor vs BeanPostProcessor + 啟動預熱陷阱

### 面試問題

`BeanFactoryPostProcessor` 同 `BeanPostProcessor` 有咩分別？如果喺 `@PostConstruct` 入面調自己嘅 `@Transactional` 方法做緩存預熱，事務會唔會生效？

### 答法

- **兩個擴展點，時機唔同**：
  - `BeanFactoryPostProcessor`（BFPP）：**Bean 未實例化之前**，改 **BeanDefinition**（設計圖）。例子：`PropertySourcesPlaceholderConfigurer` 解析 `${}`、`ConfigurationClassPostProcessor`（它係 `BeanDefinitionRegistryPostProcessor`）掃 `@Configuration`/`@ComponentScan`/`@Import` 註冊更多 BeanDefinition。
  - `BeanPostProcessor`（BPP）：**每個 Bean 實例化 + 注入之後**，改 **Bean 實例**。例子：`AutowiredAnnotationBeanPostProcessor` 處理 `@Autowired`、`CommonAnnotationBeanPostProcessor` 處理 `@PostConstruct`、`AbstractAutoProxyCreator` 喺 `postProcessAfterInitialization` 包 AOP 代理（事務、`@Async`）。
  - 一句記：**BFPP 改圖紙，BPP 改成品**。
- **`@PostConstruct` 調自己 `@Transactional` 方法**：**唔生效**。
  - `@PostConstruct` 喺 `postProcessBeforeInitialization` 階段跑，代理要到 **After** 先包；而且 `this.xxx()` 本身就唔經代理。
  - 改法：預熱放到 `ApplicationRunner` / `@EventListener(ApplicationReadyEvent.class)`（容器全部就緒、代理已經包好），並經由注入嘅另一個 Bean（或 `TransactionTemplate`）調用。
- **`@PostConstruct` 做重活嘅風險**：拖慢啟動、K8s readiness 未 ready 前就打外部依賴、拋異常會令整個 context 啟動失敗。

### 常見追問 / 陷阱

- **BFPP 用 `@Bean` 聲明要 `static`**：否則要先實例化成個 `@Configuration` 類，令該類入面嘅 `@Value`/`@Autowired` 喺 BFPP 階段處理唔到（Spring 會 log 警告）。
- **BPP 自己都係 Bean，會好早被創建**；佢依賴嘅 Bean 會被提前創建，可能錯過其他 BPP 處理。
- `SmartInitializingSingleton.afterSingletonsInstantiated()`：所有單例創建完先回調，比 `@PostConstruct` 晚、比 `ApplicationReadyEvent` 早。
