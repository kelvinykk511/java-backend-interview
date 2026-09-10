# equals 與 hashCode 契約

## 面試問題

為什麼重寫 `equals` 一定要重寫 `hashCode`？契約是什麼？

## 答法

契約（核心兩條）：

1. **`equals` 為 true → `hashCode` 必須相同**
2. `hashCode` 相同 → `equals` 不一定 true（允許碰撞）

`HashMap` / `HashSet` 先用 hash 定位桶，再用 `equals` 比內容。只改 `equals`、忘了 `hashCode` → 邏輯相等的兩個物件可能進不同桶，`contains` / `get` 失敗。

實作要點：

- 用「業務相等」的欄位一起參與兩邊（IDE / Lombok `@EqualsAndHashCode` 可生成，但要懂）
- 可變物件當 key 很危險：改了參與 hash 的欄位會「丟」在 map 裡
- 對稱、反射、傳遞、一致性；`null` 回 false；別跟繼承搞混（常見用 `getClass()` vs `instanceof` 爭議）

## 常見追問 / 陷阱

「兩個物件 `hashCode` 一樣，放進 HashMap 會怎樣？」

→ 進同一桶，再用 `equals` 區分；Java 8+ 鏈太長可能轉紅黑樹。碰撞多只影響效能，不該破壞正確性——正確性靠 equals/hashCode 契約。
