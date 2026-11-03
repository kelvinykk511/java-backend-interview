# Java 泛型：類型擦除、橋接方法、通配符 PECS

## 面試問題

Java 泛型係「偽泛型」係咩意思？類型擦除會帶嚟咩限制？`List<? extends Number>` 同 `List<? super Integer>` 點揀？

## 答法

- **類型擦除**：泛型只喺編譯期檢查，編譯後 `List<String>` 同 `List<Integer>` 都係 `List`，類型參數擦成上界（冇寫就係 `Object`，`<T extends Comparable<T>>` 擦成 `Comparable`），讀取時編譯器自動插 cast。目的係兼容 Java 5 之前嘅舊字節碼。
- **由此而嚟嘅限制**：
  - 唔可以 `new T()`、`new T[]`、`instanceof List<String>`。
  - 基本類型唔可以做類型參數（`List<int>` 唔得）→ 要裝箱，有性能成本。
  - `List<String>` 同 `List<Integer>` 嘅 `getClass()` 相等；唔可以靠泛型參數重載（`void f(List<String>)` 同 `void f(List<Integer>)` 擦除後簽名衝突）。
  - 靜態字段 / 靜態方法唔可以用類嘅類型參數 `T`。
- **橋接方法（bridge method）**：`class A implements Comparable<A> { int compareTo(A o) }` 擦除後接口方法係 `compareTo(Object)`，編譯器自動生成 `compareTo(Object)` 去 cast 再調 `compareTo(A)`，維持多態。反射 `getDeclaredMethods()` 會見到佢（`isBridge()`）。
- **PECS（Producer Extends, Consumer Super）**：
  - `? extends Number`：只讀（生產者），可以攞出 `Number`，唔可以 add（除咗 null）。
  - `? super Integer`：可寫入 `Integer`（消費者），讀出嚟只係 `Object`。
  - 例：`Collections.copy(List<? super T> dest, List<? extends T> src)`。

## 常見追問 / 陷阱

- 「擦除咗，點解 Jackson / Gson 仲可以反序列化 `List<Order>`？」→ 類、字段、方法簽名上嘅泛型信息會保留喺 class 文件嘅 `Signature` 屬性；用匿名子類 `new TypeReference<List<Order>>(){}` 將類型寫入父類簽名，再用 `getGenericSuperclass()` 讀返。運行時對象本身冇泛型信息，但聲明有。
- **堆污染**：用原始類型 `List raw = new ArrayList<String>(); raw.add(1);` 編譯只有警告，到讀取時先喺插入嘅 cast 位爆 `ClassCastException` —— 離真正出錯嘅位好遠，難查。
- `List<Object>` 唔係 `List<String>` 嘅父類（泛型唔協變），但數組係協變（`Object[] a = new String[1]; a[0]=1;` 運行時 `ArrayStoreException`）。
