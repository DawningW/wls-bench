## 结论

- **能否内联**：`set` 和 `set2` 都声明为 `inline fun`，两者在 Kotlin 编译器（`kotlinc`）阶段**都能内联**到调用点。
- **能否进行死代码优化（DCE）**：**`set` 可以，`set2` 不可以**（具体发生在 Android 的 **R8** 字节码优化阶段以及 ART/JVM 的 **JIT** 编译阶段；`kotlinc` 本身对两者都不会直接裁剪分支）。

这往往与直觉相反——很多人会误以为 `reified T` 配合 `T::class` 能在编译期常量折叠，但实际上在 Kotlin/JVM 字节码层面恰恰相反。

---

## 原因分析

### 1. 为什么 `set` 可以进行死代码优化

`set` 使用 `when (value) { is ... }`：

1. **内联后的字节码**：`kotlinc` 将 `set` 内联到调用点时，会把 `when (value)` 展开为一系列 JVM `instanceof`（`INSTANCEOF`）检查，例如：
   ```java
   Object value$iv = Integer.valueOf(123); // 或 String 等具体类型
   if (value$iv instanceof Boolean) {
       mmkv.encode(key, ((Boolean) value$iv).booleanValue(), expire);
   } else if (value$iv instanceof Integer) {
       mmkv.encode(key, ((Number) value$iv).intValue(), expire);
   } else if (...) { ... }
   ```
2. **R8 / JIT 的静态类型分析与分支裁剪**：
   - 当调用点传入的是具体类型（例如 `Int`、`Long`、`String`、`Boolean`、`ByteArray` 等 `final` 类型，或具体的 `Parcelable` 实现类）时，R8 能静态确定 `value$iv` 的精确类型（以及非空性）。
   - 对于 `Integer` 实例：`instanceof Boolean` 恒为 `false`，`instanceof Integer` 恒为 `true`。
   - R8 会将 `instanceof` 常量折叠，**直接消除所有不可达的 `else if` 和 `else` 分支（Dead Code Elimination）**，只保留命中的那一条 `mmkv.encode(...)` 调用，并可进一步消除冗余的类型强转甚至装箱开销。
   - （注：如果调用点传入的静态类型本身就是 `Any` 等宽泛类型，R8 无法静态推断具体类型，则不会裁剪分支。）

### 2. 为什么 `set2` 无法进行死代码优化

`set2` 使用 `when (T::class) { Boolean::class -> ... }`：

1. **内联后的字节码**：`kotlinc` **不会**在编译期对 `when (T::class)` 做常量折叠。内联后，每个 `X::class` 都会编译为 `Reflection.getOrCreateKotlinClass(...)` 调用，而 `when` 的分支匹配会编译为链式的 `Intrinsics.areEqual(...)` 调用：
   ```java
   KClass var$iv = Reflection.getOrCreateKotlinClass(Integer.TYPE);
   if (Intrinsics.areEqual(var$iv, Reflection.getOrCreateKotlinClass(Boolean.TYPE))) {
       mmkv.encode(key, ((Boolean) value).booleanValue(), expire);
   } else if (Intrinsics.areEqual(var$iv, Reflection.getOrCreateKotlinClass(Integer.TYPE))) {
       mmkv.encode(key, ((Integer) value).intValue(), expire);
   } else if (...) { ... }
   ```
2. **无法被 R8 / JIT 静态折叠**：
   - `Reflection.getOrCreateKotlinClass(...)` 会通过 `ReflectionFactory` 构造/返回 `ClassReference` 对象。
   - `Intrinsics.areEqual(a, b)` 会虚分派到 `ClassReference.equals(...)`（内部还会对基本类型做 `javaObjectType` 的映射查找）。
   - 这些调用对 R8 而言不是可静态求值的常量表达式，因此 **R8 无法判定各个分支的真假，所有分支都会被完整保留在最终字节码/DEX 中**，无法进行死代码消除，且在运行时还会带来额外的 `KClass` 获取与 `equals` 比较开销。

---

## 补充：`set2` 还存在功能性 Bug

除了无法做死代码优化，`set2` 在 `Parcelable` 场景下还有严重的语义问题：

- `Parcelable` 是一个接口。当调用 `set2("user", user, 0)`（其中 `class User : Parcelable`）时，编译器推断的 `T` 是 `User` 而不是 `Parcelable`。
- 此时 `T::class` 是 `User::class`，与 `Parcelable::class` 并不相等（`User::class == Parcelable::class` 为 `false`），会直接落入 `else -> false` 导致写入失败（除非调用方显式写成 `set2<Parcelable>(...)`）。
- 同理，如果实参静态类型是 `Any`（运行时是 `Int`），`set2` 也会因为 `Any::class != Int::class` 而失败。
- 而 `set` 使用 `is Parcelable`（`instanceof`），能够正确匹配所有 `Parcelable` 子类以及多态实参。

---

## 总结

- **选择 `set`**：两者都能内联，但**只有 `set` 能在 R8（及 JIT）阶段通过 `instanceof` 静态类型分析实现死代码优化**，且在 `Parcelable` 子类匹配上语义正确。
- （工程建议：如果不需要受限于 KMP 单个泛型 `expect` 签名，直接为各支持的类型提供独立重载函数 `set(key, value: Int, expire)` 等，甚至无需 `when` 和基本类型装箱，性能与包体积更优。）
