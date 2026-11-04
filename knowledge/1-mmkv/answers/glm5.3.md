**`set`（用 `is` 判断的那个）能被内联并做死代码消除，`set2` 基本不能。**

原因在于 `reified T` 内联后，编译器对两种写法的处理完全不同：

**`set`：`is` 类型检查可被常量折叠**

内联时 `T` 被替换为调用处的具体类型，比如 `set("k", "hello", 10)`：

```kotlin
when (value) {
    is Boolean -> ...  // String is Boolean → 编译期即知为 false，整个分支删除
    is Int -> ...      // 同上，删除
    ...
    is String -> mmkv.encode(key, value, expire)  // 恒真，条件判断消除
}
```

`is` 检查是语言层面的类型判断，编译器（Kotlin 前端 + R8/ProGuard）在类型已知时直接折叠为 `true`/`false`，其余分支成为死代码被移除。最终生成的代码等价于直接调用 `mmkv.encode(key, value, expire)`，没有任何运行时类型分发。

**`set2`：`T::class` 比较不会被折叠**

`when (T::class)` 本质是运行时的 `KClass.equals()` 调用。即使 `T` 被实体化为 `String`，`String::class == Boolean::class` 在编译器眼里只是两个 `KClass` 对象的 `equals` 比较——这不是语言级的类型检查，编译器不会将其折叠为常量。所以内联后：

- 8 个 `KClass` 比较（`equals` 调用）全部保留在运行时；
- 分支无法消除，`as` 强转也全部保留；
- 还额外引入了 `value as Boolean` 这类若真执行会抛 `ClassCastException` 的隐患（虽然走到不了，但验证器/优化器不一定能证明）。

另外 `set2` 还有一个逻辑 bug：`Parcelable::class` 分支几乎永远匹配不上，因为具体类型（如 `Bundle::class`）不等于 `Parcelable::class` 本身；而 `is Parcelable` 是判断实现关系，能正确命中。

**结论**：想让 reified + inline 达到零开销分发，必须用 `is` 检查（或 `when(value)` 这种智能转换友好的形式），而不是 `T::class` 比较。`set` 是正确写法。
