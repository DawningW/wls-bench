# 1-mmkv 静态分析解答

## 结论

1. **内联能力**：`set` 和 `set2` 都声明为 `inline fun <reified T>`，**两者在调用点都能被内联**。
2. **死代码优化（DCE）能力**：**`set2` 更容易且更彻底地进行死代码消除**（尤其是在 R8/ProGuard、Kotlin/Native LLVM 等静态优化器中）。

---

## 原因分析

### 1. `set2`：`when (T::class)` 基于编译期常量类型引用
- `T` 是 `reified` 泛型参数，内联到调用点时，`T::class` 会被替换为具体的编译期常量 `KClass`（例如调用 `set2("k", 123, 0)` 时，`T::class` 变为 `Int::class`）。
- `when (T::class)` 展开后等价于一系列常量比较（如 `Int::class == Boolean::class`、`Int::class == Int::class` 等）。
- 编译器/优化器（如 R8、LLVM、JIT）可以静态确定唯一命中的分支，将其余不可达分支全部作为死代码消除（Dead Code Elimination），最终只保留对应的 `mmkv.encode(...)` 调用。

### 2. `set`：`when (value)` 基于运行期类型检查（`is` / `instanceof`）
- `when (value)` 检查的是变量 `value` 在运行时的实际类型（JVM 字节码层面为 `INSTANCEOF` 指令）。
- 当静态类型 `T` 为非 `final` 类型（如 `Any`、接口、或可空类型）时，编译器无法在编译期完全排除其他分支，必须保留运行期类型判断与对应分支代码。
- 即使对于部分 `final` 类型（如 `String`、`Int`），优化器也必须依赖更复杂的流敏感类型传播（flow-sensitive type propagation）才能尝试裁剪分支，不如 `set2` 的常量折叠直接、稳定。

---

## 补充注意（语义差异）
虽然 `set2` 更有利于内联后的死代码消除，但它与 `set` 存在语义差异：
- **子类型匹配**：`set` 中的 `is Parcelable` 可以匹配 `Parcelable` 的任意实现类；而 `set2` 中的 `when (T::class)` 只做精确类型匹配，若 `T` 为具体的 `MyParcelable` 子类，`T::class == Parcelable::class` 为 `false`，会落入 `else -> false`。
- **可空性**：若传入可空类型（如 `T = Int?` 且 `value = null`），`set2` 中 `T::class` 仍为 `Int::class`，执行 `value as Int` 会抛出 `NullPointerException`；而 `set` 中 `null is Int` 为 `false`，会安全落入 `else -> false`。
