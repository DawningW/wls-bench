# 结论

1. **内联能力**：`set` 和 `set2` 都可以被 Kotlin 编译器内联（两者均声明为 `actual inline fun <reified T>`）。
2. **死代码优化（DCE）能力**：
   - **在 Kotlin 编译器（`kotlinc`）层面**：**两者都不能**由 `kotlinc` 在内联阶段直接消除死分支。
   - **在下游优化器（Android R8 / ProGuard、ART / JVM JIT）层面**：**`set` 能够进行死代码优化**（针对具体已知类型），而 **`set2` 通常无法进行死代码优化**。
   - **彻底避免运行时分支的方式**：若希望在编译期完全消除分支与装箱开销，应使用**按类型重载的函数**（如 `set(key, value: Int, ...)`、`set(key, value: Parcelable, ...)`），而不是 `inline + reified + when`。

---

# 详细原因分析

## 1. 为什么 `kotlinc` 自身无法对两者做死代码消除？

- **`set`（`when (value)`）**：
  - 函数体中的 `is Boolean`、`is Int` 等本质上是**运行时类型检查**（JVM 上编译为 `instanceof` 指令），并不依赖 `reified T`。
  - 即使调用处传入 `Int` 字面量，内联时由于 `value` 的形参类型是泛型 `T`，在 JVM 上会先触发基本类型装箱（如 `Integer.valueOf`），`kotlinc` 会把整套 `instanceof` 分支原样内联到调用处，不会在编译期裁剪分支。
- **`set2`（`when (T::class)`）**：
  - 虽然使用了 `reified T`，但 `when (T::class)` 在 Kotlin/JVM 字节码中会降级为 `Reflection.getOrCreateKotlinClass(...)` 与 `Intrinsics.areEqual(...)` 的方法调用链。
  - `kotlinc` 不会在内联时对 `KClass` 对象的 `Intrinsics.areEqual` 做常量折叠，因此所有分支同样会被保留到字节码中。

---

## 2. 为什么下游优化器（R8 / JIT）能优化 `set` 而难优化 `set2`？

- **`set` 可被 R8 / JIT 优化**：
  - `when (value)` 编译为直接的 JVM `instanceof` 指令，没有反射或 `Intrinsics.areEqual` 包装。
  - 当调用处实参是编译期可确定的 `final` 类型（如 `Boolean`、`Int`/`Integer`、`Long`、`Float`、`Double`、`String`、`ByteArray`）时，R8 的类型传播优化和 ART/JVM JIT 可以静态判定 `Integer instanceof Boolean` 恒为 `false`、`Integer instanceof Integer` 恒为 `true`，从而将不可达分支作为死代码消除。
  - （注：若调用处实参的静态类型是 `Any` 或非 `final` 的接口/基类如 `Parcelable`，由于运行期实际类型不确定，`instanceof` 分支仍会保留。）
- **`set2` 难以被 R8 优化**：
  - `Reflection.getOrCreateKotlinClass(T::class.java)` 和 `Intrinsics.areEqual(...)` 对 R8 而言是带副作用/不透明的运行时方法调用，R8 无法在编译期对 `KClass` 实例的相等性进行常量求值，因此 `when (T::class)` 的所有分支和反射元数据查询都会保留在最终产物中。

---

## 3. 额外注意：`set2` 存在严重的类型语义缺陷

`when (T::class)` 判断的是**精确类相等**（`==`），而不是**子类型关系**（`is`）：
- 当传入自定义 `Parcelable` 实现类（例如 `T` 为 `UserParcelable`）时，`UserParcelable::class == Parcelable::class` 为 **`false`**，导致 `set2` 直接走入 `else -> false`，无法正确编码。
- 当实参静态类型为 `Any`（如 `val x: Any = 123; set2("k", x, 0)`）时，`T::class` 为 `Any::class`，同样会命中 `else -> false`。
- 而 `set` 使用 `is Parcelable` / `is Int` 进行运行时检查，在语义上是正确的。
