# Spring Boot：自動配置與 @Conditional

## 面試問題

Spring Boot「自動配置」是怎麼生效的？`@ConditionalOnClass` / `OnMissingBean` / `OnProperty` 各解決什麼？怎麼 debug「starter 引了卻沒 Bean」？

## 答法

**機制（面試版）**

1. `@SpringBootApplication` 含 `@EnableAutoConfiguration` → 啟動時載入自動配置清單。
2. 清單來源：Boot 2.7+ 用 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`；舊版是 `META-INF/spring.factories`（`EnableAutoConfiguration` 鍵）。
3. 每個 `*AutoConfiguration` 上掛一堆 **`@Conditional*`**：條件成立才註冊裡面的 `@Bean`。
4. 你自己的 `@Configuration` / `@Bean` 通常優先；Boot 用 `OnMissingBean` 做到「有自訂就不裝預設」——**自動配置不跟用戶 Bean 搶**。

**常見條件（記名字就夠）**

| 註解 | 意思 |
|------|------|
| `@ConditionalOnClass` | classpath 有某類才啟用（有沒有依賴） |
| `@ConditionalOnMissingBean` | 容器裡還沒有同型／同名 Bean 才建預設 |
| `@ConditionalOnProperty` | 某個 `application.yml` 屬性匹配才開 |
| `@ConditionalOnWebApplication` | Web 環境才裝 |

**覆蓋／關閉**

- 自己宣告同型 `@Bean`（觸發 `OnMissingBean` 讓路）
- `spring.autoconfigure.exclude=…` 排除整個 AutoConfiguration
- 改對應 `spring.xxx.enabled=false`（若該模組有提供）

**怎麼 debug**

- 啟動加 `--debug`（或 `debug=true`）→ 印 **ConditionEvaluationReport**：哪些 auto-config matched / negative matched。
- 對照 exclude、屬性、自己是否已提供同型 Bean。

**面試一句話**：自動配置 = **條件化的預設 Bean 工廠**；條件不滿足就不裝，有你的 Bean 就讓路。

## 常見追問 / 陷阱

「classpath 有類（OnClass 過了）為什麼還是沒有那個 Bean？」

→ OnClass 只是門檻之一。還可能：`OnProperty` 沒開、`OnMissingBean` 被你（或別的 starter）的同型 Bean 擋掉、被 `exclude`、或 `@Bean` 方法上還有更細的條件沒過。看 Report，別只看依賴樹。

「`@Conditional` 寫的順序／放類上還是方法上有差嗎？」

→ 有。類級條件不過 → 整個配置類跳過；方法級只影響該 `@Bean`。條件評估有階段（parse vs register），自訂條件別假設「一定看得到別的 Bean」。

「自訂 starter 常見坑」

→ `AutoConfiguration.imports`（或 factories）路徑／類名寫錯；忘記加 `@AutoConfiguration`／條件；把業務 `@Component` 掃進自動配置包導致重複或循環；用戶沒引對依賴 → OnClass 永遠不過。

## 小練習

引了 `spring-boot-starter-data-redis`，卻沒有 `RedisTemplate` Bean。你會先查哪三件事？`--debug` 報告裡 matched / negative 分別代表什麼？
