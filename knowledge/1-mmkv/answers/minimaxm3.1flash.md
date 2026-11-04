我先加载一下图表设计规范，然后给你拆解这两段代码的编译期行为。
## 结论

**`set` 能被内联并进行死代码消除；`set2` 虽然也能内联，但分支消不掉。**

关键差别不在 `inline`/`reified`（两者都一样），而在于**分支的判据**：`set` 用 `is` 判**值**，`set2` 用 `T::class` 判**类型参数**。

## 为什么 `set` 能被优化掉

内联发生在调用点，此时 `T` 已被替换为具体类型，`value` 的静态类型随之确定（比如 `Int`）。于是：

- `is Boolean ->` 变成编译期常量 `false`，`is String ->`、`is Long ->` 等同理 → 这些分支在 IR 优化阶段被常量折叠 + 死分支删除，**整块代码从字节码里消失**。
- 命中的 `is Int ->` 分支里 `value` 被 smart cast 成 `Int`，`mmkv.encode(key, value, expire)` 精确解析到 `encode(String, Int, Int)` 重载。
- 泛型擦除带来的装箱也一并消失：调用点传的是 `Int`，最终只有一次 `encode` 调用。

补充一个前提：`reified T` 要求调用点的 `T` 必须是具体类型，否则编译器直接报错（"Cannot use 'T' as reified type parameter"）。所以**所有合法调用点上 `T` 都是已知的**，`set` 的 DCE 必然生效。

## 为什么 `set2` 不行

`T::class` 不是编译期常量。即使 `T` 已知为 `Int`：

- 在字节码里它会编译成运行时取值（类似 `Reflection.getOrCreateKotlinClass(Integer.class)`），得到一个 `KClass` 对象。
- `when (T::class)` 对 `KClass` 做的是对象相等比较，编译器**无法**把它折叠成常量选择 → 8 个分支（8 个不同重载的 `encode` 调用）全部保留在生成的代码里。
- `value as Boolean` 等变成运行时 `checkcast` + 拆箱，无法消除。

也就是说，`set2` 每次调用都要付一次 KClass 查找 + 一串 `equals` 比较的开销。

## 顺带两个正确性问题

`set2` 除了慢，还有语义风险：

|场景 | `set` | `set2` |
|---|---|---|
| `T = Int?`，值为 `null` | `is Int` 不成立 → 落到 `else`，返回 `false`，行为可预期 | `T::class` 仍是 `Int::class` → 走 `Int` 分支 → `value as Int` 对 `null` 抛 NPE / TypeCastException |
| 类型不匹配 | `is` 检查天然安全 | `as` 是 unchecked cast，靠分支保证，一旦判据与实际不符就崩 |

所以建议**保留 `set`、删掉 `set2`**。

## 想让 `set2` 也优化掉？

只能换判据，而 `when (value) { is X -> }` 本身正是 `set` 的写法——这正是 `is` 判据的优势：既能被编译器折叠掉多余分支，又天然免于 `ClassCastException`。

另外提醒一点：多平台下 `expect` 侧也必须写成 `inline fun <reified T>`，且 `actual` 必须同为 `inline` + `reified`，否则 `actual inline` 声明会编译报错。
