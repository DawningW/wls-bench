严格来说：**两个函数都声明了 `inline`，所以都会被内联。**

但如果问题是“哪个能基于 `reified T` 在编译期裁剪 `when` 分支、做死代码优化”，答案是：**`set2`**。

- `set` 用的是：

```kotlin
when (value) {
    is Boolean -> ...
    is Int -> ...
}
```

这是对 **运行时对象** 做 `instanceof` 类型判断。即使内联后，编译器通常也不能确定 `value` 一定是某个具体类型，所以这些分支一般会保留，不能可靠做死代码消除。

- `set2` 用的是：

```kotlin
when (T::class) {
    Boolean::class -> ...
    Int::class -> ...
}
```

它基于 `reified T`。内联到调用点后，`T` 会被替换成真实类型，例如：

```kotlin
set2<Int>("key", 1, expire)
```

展开后近似于：

```kotlin
when (Int::class) {
    Boolean::class -> ... // 恒为 false，可视为死代码
    Int::class -> ...     // 只有这个分支可达
    ...
}
```

所以 **`set2` 才是那个能利用 reified 类型进行类型分派并具备死代码优化条件的写法**。

但要注意语义差别：`set2` 是**精确类型匹配**，不像 `set` 的 `is Parcelable` 那样支持子类。例如一个自定义 `data class User(...) : Parcelable`，调用 `set2<User>(...)` 不会命中 `Parcelable::class` 分支，而 `set<User>(...)` 会命中 `is Parcelable`。
