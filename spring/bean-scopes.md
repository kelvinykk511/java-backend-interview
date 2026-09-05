# Spring Bean Scopes

## 面試問題

Spring Bean 預設 scope？`prototype` 每次拿都新建嗎？

## 答法

- 預設 **`singleton`**（一個 Spring 容器內一份）。
- **`prototype`**：每次 `getBean` 新建。
- 若被 singleton 注入，通常**只會拿到一次**（注入時創建）。要每次新實例可用 `ObjectFactory` / `@Lookup`。

## 常見追問

還有哪些 scope？→ `request` / `session` / `application`（Web 環境）。
