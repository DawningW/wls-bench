**结论：`set2`（`when (T::class)`）能被内联并触发死代码消除；`set`（`when (value)`）虽然同样会内联，但分支基本删不掉。**

先说清楚一件事：内联本身两者都会发生，区别在**内联展开之后**，剩下的 `when` 条件是不是编译期常量。
## 为什么

`reified T` 在调用点会被**具体类型实参替换**。`set2<String>("k","v",0)` 内联后大致变成：

```kotlin
when (String::class) {
    Boolean::class  -> ...   // 常量 == 常量 → false
    Int::class      -> ...   // → false
    String::class   -> mmkv.encode(key, value as String, expire)  // → true
    else            -> ...
}
```

所有条件都是编译期可判定的常量比较，`value as String` 这个强转在 `T` 已知后也变成空操作。剩下的就是「常量折叠 → 剪掉不可达分支 → 连带删掉那些 `encode` 重载调用」，这就是死代码消除。

`set` 内联后是：

```kotlin
val v: Any? = value
when (v) {
    is Boolean -> ...   // 运行时 instanceof
    is Int     -> ...   // 编译器不能假定 v 不是 Int
    ...
}
```

`is` 依赖的是**运行时值**，不是类型参数。即便内联时 `T` 已知，Kotlin 前端也不会反过来用「T = String」去证明 `value is Int` 恒假（value 可能为 null、可能是子类型、也可能来自反射），所以 9 个分支连同 8 个 encode 重载都会留在产物里。一个例外是 Kotlin/Native：它对 final 类的类型检查有较强的静态判定，偶尔也能剪掉一部分，但 `is Parcelable` 这类接口检查剪不掉，且远不如 `T::class` 可预测。

## 优化真正生效的前提

- **Kotlin 前端不删分支**：JVM 上用 `javap` 看，两者都是一串 `if/else`。区别只是 `set2` 的条件是可折叠的 KClass 常量比较。
- **要靠后端优化器**：Android 需 release + R8（`minifyEnabled`，R8 有 Kotlin KClass 常量的专门折叠）；Kotlin/Native 靠 LLVM 的常量折叠 + DCE；Kotlin/JS 靠 IR 后端的 DCE。debug 构建两者差别不大。
- **T 必须是具体类型**：通过函数引用 `::set2`、反射调用，或 JVM 上被 Java 调用时，编译器会保留一份 `T` 擦除成 `Object` 的普通方法体，那份代码无法折叠。KMP 里 `actual inline` 同样会为二进制依赖生成这样一份公开方法。

## `set2` 有两个真实 bug

1. **`Parcelable::class` 是精确相等，不是子类型检查**。`T = User::class` 或 `Bundle::class` 时，`T::class == Parcelable::class` 为 false → 落进 `else -> false`，值根本写不进去。而 `is Parcelable` 是对的。
2. **null 行为不同**：`T = String?` 且 `value == null` 时，`T::class` 仍是 `String::class`，`value as String` 直接 NPE；`set` 的 `is String` 不匹配 null，会安全走 `else`。

## 建议

最干净的做法是**别用泛型 `when`，直接写重载**，让编译期静态分发，零分支、Parcelable 子类自动匹配、也不需要 reified：

```kotlin
fun set(key: String, value: Boolean, expire: Int) = mmkv.encode(key, value, expire).toUnit()
fun set(key: String, value: String, expire: Int) = mmkv.encode(key, value, expire).toUnit()
fun set(key: String, value: Parcelable, expire: Int) = mmkv.encode(key, value, expire).toUnit()
```

如果 API 必须保留 `inline <reified T>` 签名，用 `T::class` 处理基础类型，`else` 里做可空安全兜底（对基础类型调用点，`else` 一样会被常量折叠掉）：

```kotlin
else -> (value as? Parcelable)?.let { mmkv.encode(key, it, expire) } ?: false
```

一句话取舍：**要优化选 `set2`，要语义正确选 `set`；两者都想要就用重载。**
