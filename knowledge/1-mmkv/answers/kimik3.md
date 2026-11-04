**`set2` 能真正实现编译期内联 + 死代码消除，`set` 不能。**

原因：

**`set`（`when (value)`）— 运行时判断**
- `when (value)` 是对**运行时值**的类型检查（`is Boolean` 等）。
- 虽然函数标了 `inline` + `reified`，但分支选择发生在运行时，编译器无法在编译期确定走哪个分支。
- 内联后所有分支代码都会保留在调用处，只是省了函数调用开销，没有死代码优化。

**`set2`（`when (T::class)`）— 编译期判断**
- `T::class` 依赖 `reified T`，内联时编译器知道 `T` 的具体类型（比如调用 `set2<Int>("k", 1, 0)` 时 `T::class == Int::class`）。
- 编译器在编译期就能把 `when` 折叠成唯一命中的那个分支，其余分支作为死代码直接消除。
- 内联 + DCE 之后，调用点实际只生成一行 `mmkv.encode(key, value as Int, expire)`，连 `when` 本身都不会留在字节码里。

**补充两点：**

1. `Parcelable::class -> ...` 这个分支在 `set2` 里基本匹配不到——`T::class` 是具体类（如 `MyData::class`），不会等于接口 `Parcelable::class`。如果想支持 Parcelable 子类型，得用 `Parcelable::class.java.isAssignableFrom(T::class.java)`，但那就退回运行时判断、失去 DCE 了。`set` 用 `is Parcelable` 反而能正确处理子类型。
2. 所以实践建议：**基本类型用 `set2` 拿编译期优化；`Parcelable` 单独写一个非 reified 的重载**（如 `fun <T : Parcelable> set(key, value, expire)`），既保留类型安全又不浪费内联。

验证方式很简单：编译后看字节码/反编译，`set2<Int>` 的调用点只剩一条 `encode` 调用，而 `set` 的调用点能看到完整的类型判断链。
