**结论：**

**`set2`**（使用 `when (T::class)` 的版本）能够在 **Kotlin 编译期**实现真正的死代码消除（Dead Code Elimination, DCE）。

但是，**`set2` 在你的代码中存在致命的逻辑缺陷**（特别是对于 `Parcelable`），在实际业务中你可能必须使用 `set`。

以下是深度的原理解析以及为什么你需要小心：

---

### 1. 为什么 `set2` 能做到死代码消除？

在 `set2` 中，你使用了具体化类型参数（`reified T`）和类引用（`T::class`）。
因为函数是 `inline` 的，Kotlin 编译器在调用点会进行**编译期展开**。

如果你这样调用：`set2("age", 25, 0)`
Kotlin 编译器在编译时明确知道 `T` 就是 `Int`。它会将 `when (T::class)` 直接替换为 `when (Int::class)`，在编译阶段它就能算出只有 `Int::class` 这一分支为真。

**编译后的等效字节码（调用处）：**

```kotlin
// 其他所有的 Boolean, Long, String 等分支都会在编译期被彻底抛弃（死代码消除）
mmkv.encode("age", 25 as Int, 0)

```

这种写法实现了“零开销”抽象，生成的字节码极其干净，没有多余的判断逻辑。

### 2. 为什么 `set` 不能在编译期完全消除？

在 `set` 中，你使用了 `when (value)` 配合 `is` 关键字。
`is` 关键字在底层对应的是 Java 的 `instanceof` 指令，这是一个运行时（Runtime）的类型检查。

如果你调用：`set("age", 25, 0)`
虽然函数被内联了，但 Kotlin 编译器在前端处理时，通常会原封不动地把这些 `instanceof` 判断搬到调用点。

**编译后的等效字节码（调用处）：**

```kotlin
val value = 25
if (value instanceof Boolean) { ... }
else if (value instanceof Integer) { mmkv.encode("age", value, 0) }
// ... 后续的 instanceof 判断依然存在

```

虽然现代的 R8 / ProGuard 或者 JVM 的 JIT（即时编译器）在后期优化时大概率也能把这些死代码优化掉，但这依赖于后期的优化器，**Kotlin 编译器本身**不会在编译期为你剔除这些 `is` 分支。

---

### ⚠️ 致命陷阱：为什么你可能仍需要使用 `set`？

虽然 `set2` 在性能优化上更胜一筹，但你的代码中包含了一个接口类型：`Parcelable`。这里会产生一个严重的 Bug。

* **`set2` 的行为（类精准匹配）：** `when (T::class)` 比较的是具体的类。如果你传入一个实现了 `Parcelable` 的数据类 `User`，此时 `T::class` 是 `User::class`。`User::class == Parcelable::class` 是 **`false`**！因此，代码会走到 `else -> false`，导致保存失败。
* **`set` 的行为（支持多态/继承）：** `when (value) { is Parcelable -> ... }` 检查的是 `value` 是否属于 `Parcelable` 的实例（相当于 `instanceof Parcelable`）。传入 `User` 会返回 **`true`**，代码能正确执行。

### 总结建议

如果你需要处理 **多态、接口或继承体系**（比如 `Parcelable`），你**必须**使用 `set`。不要为了微小的死代码消除牺牲正确性。

如果你确信只处理 **精确的基础数据类型**（Int, String, Float, Boolean 等且无继承场景），那么 `set2` 是性能和字节码最优的选择。
