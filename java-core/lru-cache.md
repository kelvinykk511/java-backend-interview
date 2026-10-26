# 手寫 LRU Cache（get/put O(1)）

## 面試題
手寫一個 LRU cache，容量固定，`get`/`put` 都要 O(1)。

## 答案要點
- **HashMap + 雙向鏈表**：map 負責 O(1) 搵節點；鏈表負責記住「新舊」順序。頭 = 最近用，尾 = 最耐冇用。
- `get`：map 搵到 → 節點搬去頭 → 返回；搵唔到返回 -1/null。
- `put`：已存在 → 改值 + 搬去頭；唔存在 → 新節點插頭；超容量 → 刪尾節點，**同時喺 map 刪佢個 key**（所以 Node 要存 key）。
- 用 dummy head/tail 哨兵，免晒 null 判斷。

```java
class LRUCache<K, V> {
    private class Node { K k; V v; Node prev, next; Node(K k, V v){this.k=k;this.v=v;} }
    private final int cap;
    private final Map<K, Node> map = new HashMap<>();
    private final Node head = new Node(null, null), tail = new Node(null, null);
    LRUCache(int cap) { this.cap = cap; head.next = tail; tail.prev = head; }

    public V get(K k) {
        Node n = map.get(k);
        if (n == null) return null;
        unlink(n); addFirst(n);
        return n.v;
    }
    public void put(K k, V v) {
        Node n = map.get(k);
        if (n != null) { n.v = v; unlink(n); addFirst(n); return; }
        n = new Node(k, v); map.put(k, n); addFirst(n);
        if (map.size() > cap) { Node old = tail.prev; unlink(old); map.remove(old.k); }
    }
    private void unlink(Node n) { n.prev.next = n.next; n.next.prev = n.prev; }
    private void addFirst(Node n) { n.next = head.next; n.prev = head; head.next.prev = n; head.next = n; }
}
```

- **快捷版**：`new LinkedHashMap<>(cap, 0.75f, true)`（`accessOrder=true`）+ override `removeEldestEntry` 返回 `size() > cap`。面試通常要你先講呢個，再手寫上面嗰個。

## 常見追問 / 陷阱
- **線程安全**：`get` 會搬節點 = 改結構，所以讀都係寫。用 `ReadWriteLock` 嘅讀鎖包 `get` 係錯；要全部用同一把鎖（`synchronized`/`ReentrantLock`），或者分段。
- `LinkedHashMap` accessOrder 模式下 `get` 都會改 `modCount`，邊迭代邊 `get` 會 CME。
- **生產唔好自己寫**：用 Caffeine（W-TinyLFU，命中率高過純 LRU，抗一次性掃描污染），支持過期、大小上限、異步加載。
- 忘記喺 map 刪被淘汰嘅 key → 內存洩漏，係最常見 bug。

相關：[HashMap 擴容/樹化](hashmap-resize-treeify.md)、[fail-fast/CME](collections/fail-fast-cme.md)
