# Spring Boot 啟動流程（SpringApplication.run → refresh）

## 面試問題

`SpringApplication.run()` 做了什麼？自動配置的 Bean 喺邊一步被建立？

## 答法

### `new SpringApplication`

- 推斷 web 類型（**SERVLET** / **REACTIVE** / **NONE**）
- 從 `spring.factories` 載入 `ApplicationContextInitializer`、`ApplicationListener`
- 推斷 main class

### `run()` 主線

1. RunListeners **starting**
2. 準備 **Environment**（命令行參數、`application.yml`、profile；`EnvironmentPreparedEvent`）
3. Banner
4. **createApplicationContext**
5. **prepareContext**（跑 Initializer、註冊主類 `BeanDefinition`）
6. **refreshContext** ← 核心
7. afterRefresh → **started**
8. 調用 `ApplicationRunner` / `CommandLineRunner`
9. **ready**（`ApplicationReadyEvent`）

### `refresh()` 重點

| 階段 | 做咩 |
|------|------|
| `invokeBeanFactoryPostProcessors` | `ConfigurationClassPostProcessor` 解析 `@Configuration` / `@ComponentScan` / `@Import`；自動配置喺呢步經 `AutoConfigurationImportSelector` 讀 `AutoConfiguration.imports`，按 `@Conditional` 篩選 —— **只係註冊 BeanDefinition，未建對象** |
| `registerBeanPostProcessors` | 註冊 BPP |
| `onRefresh` | Servlet 應用喺呢度建立內嵌 Tomcat（**未對外收流量**） |
| `finishBeanFactoryInitialization` | 實例化所有非 lazy 單例：構造/依賴注入 → Aware → BPP before → `@PostConstruct` / `afterPropertiesSet` → BPP after（**AOP 代理喺呢度生成**） |
| `finishRefresh` | `SmartLifecycle` start（Web server **開始監聽**）、`ContextRefreshedEvent` |

**面試一句話**：自動配置喺 **BeanFactoryPostProcessor** 階段只註冊定義；**真正建 Bean** 喺 `finishBeanFactoryInitialization`；對外收流量喺 `finishRefresh` 之後。

## 常見追問 / 陷阱

**(a) BPP 失效**

→ 喺 `BeanFactoryPostProcessor`（或過早被依賴的 Bean）注入普通 Bean 會令佢**提早實例化**，錯過 BPP → `@Transactional` / `@Async` 代理失效。日誌會見到 *"is not eligible for getting processed by all BeanPostProcessors"*。

**(b) Runner 與流量**

→ `ApplicationRunner` / `CommandLineRunner` 喺 **Web server 已監聽之後**先跑 → 預熱未完成流量可能先到。K8s readiness 應跟 `ApplicationReadyEvent` 之後的 `ReadinessState.ACCEPTING_TRAFFIC`。

**(c) 啟動慢排查**

→ `BufferingApplicationStartup` + actuator `/actuator/startup` 睇邊個 Bean 慢；`spring.main.lazy-initialization=true` 可加快但會把錯誤**推遲到第一次請求**。

## 小練習

啟動後第一個 HTTP 請求才報 `NoSuchBeanDefinitionException`，但 `--debug` 顯示 auto-config matched。你會懷疑 lazy、條件評估階段，定係 `@Import` 順序？先查哪兩個配置？
