# Spring Boot：自動配置與 @Conditional

## 面試問題

Spring Boot「自動配置」是怎麼生效的？`@ConditionalOnClass` / `OnMissingBean` / `OnProperty` 各解決什麼？怎麼關掉或覆蓋某個 AutoConfiguration？

## 答法

**機制（面試版）**

1. 啟動時載入 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`（Boot 2.7+；舊版是 `spring.factories`）。
2. 每個 `*AutoConfiguration` 類上掛一堆 **`@Conditional*`**：條件成立才註冊裡面的 `@Bean`。
3. 你自己的 `@Configuration` / `@Bean` 通常優先；Boot 用 `OnMissingBean` 做到「有自訂就不裝預設」。

**常見條件（記名字就夠）**

| 註解 | 意思 |
|------|------|
| `@ConditionalOnClass` | classpath 有某類才啟用（有沒有依賴） |
| `@ConditionalOnMissingBean` | 容器裡還沒有同型 Bean 才建預設 |
| `@ConditionalOnProperty` | 某個 `application.yml` 屬性匹配才開 |
| `@ConditionalOnWebApplication` | Web 環境才裝 |

**覆蓋 / 關閉**

- 自己宣告同名/同型 `@Bean`（配合 `OnMissingBean`）
- `spring.autoconfigure.exclude=…` 排除整個 AutoConfiguration
- 改對應 `spring.xxx.enabled=false`（若該模組有提供）

**面試一句話**：自動配置 = **條件化的預設 Bean 工廠**；條件不滿足就不裝，有你的 Bean 就讓路。

## 常見追問 / 陷阱

「為什麼我引了 starter 卻沒有某個 Bean？」

→ 先查：條件沒過（缺類、屬性沒開）、被 `exclude`、或你已有同型 Bean 觸發了 `OnMissingBean`。用 `--debug` 或 `ConditionEvaluationReport` 看哪條條件失敗，比瞎猜快。
