**`set2` 能进行死代码优化**，`set` 不能（或不可靠）。两者都能内联，但优化机制完全不同：

## 核心区别

**`set2` — `when (T::class)` 是编译期常量比较**

内联后 reified `T` 被替换为具体类型，`T::class` 变成编译期常量：

```kotlin
// 调用 set2("key", 42, 100)，T = Int
// 内联展开后等价于：
when (Int::class) {           // 编译期常量
    Boolean::class -> ...     // ❌ 编译器直接删除
    Int::class -> mmkv.encode(key, value as Int, expire)  // ✅ 唯一存活分支
    Long::class -> ...        // ❌ 删除
    // ...其余全删
}
```

编译器对常量 `when` 做常量折叠（constant folding），**其余分支直接消失**，最终只保留一个 `mmkv.encode` 调用。

**`set` — `when (value)` 是运行时 `instanceof` 检查**

内联后 `value` 虽然静态类型是 `Int`，但 `is` 检查编译成 `instanceof` 字节码：

```kotlin
// 调用 set("key", 42, 100)，T = Int
// 内联展开后等价于：
when (value) {                     // value: Int
    is Boolean -> ...              // 编译器知道不可能，但只给 warning，不删字节码
    is Int -> ...                  // 运行时 instanceof 检查
    is Long -> ...                 // instanceof 检查（冗余但未被消除）
    // ...全部分支的字节码都保留
}
```

Kotlin 编译器对 `is` 分支**不做死代码消除**，所有分支的字节码都会生成。即使 `value` 静态类型是 `Int`，`is Boolean` 分支只是产生一个编译警告，运行时靠 JIT（C2/Graal）可能间接优化，但编译产物不干净。

## 总结

| | `set` | `set2` |
|---|---|---|
| 内联 | ✅ | ✅ |
| 死代码消除 | ❌ 保留所有 `instanceof` 分支 | ✅ 只保留匹配分支 |
| 机制 | `when (value)` → 运行时类型检查 | `when (T::class)` → 编译期常量折叠 |

## ⚠️ `set2` 有个坑

`when (T::class)` 做的是**精确类匹配**，不处理继承：

```kotlin
@Parcelize data class User(...) : Parcelable

set2("key", User(...), 100)
// T::class = User::class ≠ Parcelable::class
// → 走 else 分支，返回 false，数据丢失！
```

而 `set` 的 `is Parcelable` 是 `instanceof`，子类能正确匹配。所以实际项目中通常的做法是 **`set2` 的死代码优化 + `set` 的继承语义**，用 `when` 对 `T::class` 加上 `Parcelable::class.java.isAssignableFrom(T::class.java)` 之类的兜底，或者直接用 sealed class + 编译期生成。
