# synchronized vs ReentrantLock

## 面試問題

`synchronized` 和 `ReentrantLock` 怎麼選？

## 答法

- **`synchronized`**：JVM 內建、自動釋放、可重入；適合簡單臨界區。
- **`ReentrantLock`**：可中斷、可超時試鎖、公平鎖可選、可多條件 `Condition`；記得在 `finally` 裡 `unlock`。

## 常見追問 / 陷阱

公平鎖吞吐通常更差，預設非公平就好。
