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
