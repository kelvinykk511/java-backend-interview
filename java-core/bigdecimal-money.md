# BigDecimal 與金額計算陷阱

## 面試問題

金額點解唔可以用 `double`？用咗 `BigDecimal` 仲有咩坑？

## 答法

- **`double` 唔得**：二進制浮點表示唔到 0.1，`0.1 + 0.2 = 0.30000000000000004`，累計對帳一定差數。
- **構造要用字符串**：`new BigDecimal(0.1)` 得到 `0.1000000000000000055511...`；用 `new BigDecimal("0.1")` 或 `BigDecimal.valueOf(0.1)`（行 `Double.toString`）。
- **比較用 `compareTo`，唔好用 `equals`**：`equals` 連 scale 一齊比，`1.0` 同 `1.00` equals 係 false，`compareTo` 係 0。
- **`divide` 一定要畀 scale + `RoundingMode`**：`1/3` 冇指定會拋 `ArithmeticException: Non-terminating decimal expansion`。
- **捨入方向係業務規則**：`HALF_UP`（四捨五入）vs `HALF_EVEN`（銀行家捨入，減少累計偏差）。交易所常見：向用戶收嘅手續費 `UP`，畀用戶嘅數 `DOWN`，精度按幣種/交易對配置（`setScale(precision, mode)`）。
- **輸出用 `toPlainString()`**：`toString()` 可能出科學記數法（`new BigDecimal("100").stripTrailingZeros()` → `1E+2`）。
- **跨系統**：DB 用 `DECIMAL(p,s)`（例如 `DECIMAL(36,18)`）；JSON 傳字符串，因為 JS number 只得 2^53 精度；Jackson 可開 `WRITE_BIGDECIMAL_AS_PLAIN`。
- **另一派**：用 `long` 存最小單位（分、satoshi），快又冇 scale 問題；ETH wei（18 位）會爆 `long` → 用 `BigInteger`。
- **`BigDecimal` 係不可變**：`a.add(b)` 唔會改 `a`，要接返回值。

## 常見追問 / 陷阱

「用 `BigDecimal` 做 `HashMap`/`HashSet` key 有咩問題？」

→ `hashCode` 同 `equals` 都睇 scale：`put(new BigDecimal("100.10"))` 之後用 `"100.1"` 查唔到，Set 入面仲會有兩個「同價」元素。修正：入 map 前統一 `setScale` 或 `stripTrailingZeros()`；或者用 `TreeMap`（靠 `compareTo`，`1.0` 同 `1.00` 當同一個 key）。訂單簿價格檔位通常本身就用 `TreeMap`/跳表。

相關：[equals 與 hashCode 契約](./equals-hashcode.md)
