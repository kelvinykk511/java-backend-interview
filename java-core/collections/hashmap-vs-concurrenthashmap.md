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
