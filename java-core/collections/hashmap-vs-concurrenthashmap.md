# HashMap vs ConcurrentHashMap

## 面試問題

HashMap 為什麼線程不安全？和 ConcurrentHashMap 差在哪？

## 答法

- **HashMap**：多線程下可能死循環（JDK7 擴容）、丟數據、`size` 不準；讀寫都無鎖。
- **ConcurrentHashMap（JDK8）**：CAS + `synchronized` 鎖桶頭（node）；讀大多無鎖；`size` 用計數單元近似；**不允許** `null` key/value。

## 常見追問

為什麼 ConcurrentHashMap 不允許 `null`？

→ 歧義：拿不到值時，分不清是「沒有這個 key」還是「值就是 null」。

## 小練習

為什麼 HashMap 的容量要是 2 的冪？


## 補充：HashMap 允許 null 嗎？

允許。

- `HashMap`：允許 **一個** `null` key，以及 **多個** `null` value。
- `ConcurrentHashMap`：key / value 都不允許 `null`。
- `Hashtable`：也不允許 `null`。

為什麼一邊行一邊不行？單線程下可用 `containsKey` 分辨「沒有 key」還是「值是 null」；併發下兩次查詢之間狀態可能變了，歧義解不開，所以 CHM 直接禁止。
