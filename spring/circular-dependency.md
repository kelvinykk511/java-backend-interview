# Spring 循環依賴與 @Lazy

## 面試問題

Spring 單例 Bean 之間互相注入為什麼有時能起來、有時啟動直接失敗？`@Lazy` 解決的是哪一層問題？

## 答法

- **三級緩存思路（單例）**：提前暴露「早期引用」（未完成屬性填充的工廠／物件），讓 A→B→A 在 setter／欄位注入時能先拿到對方引用，再繼續初始化。
- **構造器注入**：兩邊都用構造器互相依賴時，無法先建半成品，通常直接啟動失敗——這是面試最愛對比點。
- **`@Lazy`**：注入的是代理；真正用到才去容器拿完整 Bean，打斷「創建期」的環，不是魔法消掉設計上的環。
- **實務態度**：能重構就拆模組／抽介面／事件；臨時用 `@Lazy` 或改 setter 要說得出風險（啟動期發現不了錯誤、代理邊界）。
- **面試一句話**：單例 + 屬性注入靠三級緩存「先曝光再填滿」扛循環；構造器循環要改設計或 `@Lazy`；別把能起來當成設計正確。

## 常見追問／陷阱

「多例（prototype）循環依賴呢？」

→ Prototype 每次新建、不會像單例那樣穩定放進單例緩存，循環依賴更容易直接失敗。還有：構造器注入 + `@Autowired` 必填時，環一出現啟動就爆，比運行期 NPE 更好查。

## 小練習

A、B 互相 `@Autowired` 構造器注入啟動失敗；改成欄位注入後能起來。向面試官你會怎麼解釋差異，以及你更傾向哪種修法？

## 深入：三級緩存（2026-10-21 refresh）

**面試題**：Spring 點樣解決單例 setter/字段注入嘅循環依賴？點解要三級緩存，二級唔得？

- 一級 `singletonObjects`：完整 Bean。
- 二級 `earlySingletonObjects`：已暴露嘅早期引用（可能已經係代理）。
- 三級 `singletonFactories`：`ObjectFactory`，調 `getEarlyBeanReference()`——**有 AOP 就喺呢度提早生成代理**。
- 流程（A↔B）：A 實例化（構造完，屬性未填）→ 放 A 嘅 factory 入三級 → 填 A 屬性要 B → 建 B → B 要 A → 三級 factory 攞 A 早期引用（必要時係代理）放二級 → B 完成 → A 完成，放一級。
- **點解要三級**：正常 AOP 代理喺初始化**最後**（`postProcessAfterInitialization`）先生成；三級 factory 令到「只有真係出現循環依賴」先提早生成代理，冇循環就保持正常時機。只用二級就要每個 Bean 一實例化就生成代理，破壞生命周期設計。
- **解決唔到**：構造器注入（未實例化，冇早期引用）、prototype、`@Async` 類（代理由另一個 BeanPostProcessor 後置生成，早期引用≠最終對象 → `BeanCurrentlyInCreationException`「has been injected into other beans in its raw version」）。
- **Spring Boot 2.6+ 預設禁止循環依賴**（`spring.main.allow-circular-references=false`），啟動直接報錯。

**陷阱**：解法唔係開返 allow-circular-references，而係改設計——抽公共邏輯去第三個 Bean、用事件解耦、或者 `@Lazy` 注入代理（治標）。
