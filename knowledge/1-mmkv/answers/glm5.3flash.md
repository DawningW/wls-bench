**答案：`set2`（`when (T::class)` 那个）。**

## 原理

两个函数都标了 `inline`，所以**都能被内联**——这点没有区别。区别在内联之后分支还能不能被消掉：

**`set2`：分支条件是编译期常量。** `reified T` 的核心机制是：内联展开发生在调用点，此时 `T` 被替换为具体类型。调用 `set2("k", 100, 0)`（T 推断为 `Int`）后，内联体大致变成：

```kotlin
when (Int::class) {                    // 编译期已知的常量
    Boolean::class -> mmkv.encode(...)  // 静态不可达 → 死代码
    Int::class -> mmkv.encode(key, value as Int, expire)  // 唯一可达分支
    Long::class -> ...                  // 死代码
    ...
}
```

每个调用点只有一个分支可达，其余 7 个分支结构上是死代码，优化器可以消除（Kotlin/Native 上 LLVM 能真正做 DCE；JVM 上 kotlinc 不会折叠 `KClass.equals`，分支会留在字节码里，但会被 HotSpot JIT 很快折叠掉——不过结构上它已经是“可静态判定”的了）。

**`set`：分支条件是运行时类型。** `when (value)` 的每个分支是 `instanceof` 检查，内联后依然是真实的运行时判断：

```kotlin
when (value) {          // 运行时才决定走哪个分支
    is Boolean -> ...
    is Int -> ...
    ...
}
```

一般情形下（比如 `T = Any` 时传入 `Int` 值），每个分支都可能被走到，编译器无法消掉任何一个分支——所有 8 条分支会原样出现在每个调用点。而且 `set` 里根本没有用到 `T` 作为类型（没写 `T::class`），它的 `reified` 是多余的。

## 一个需要注意的行为差异

`set2` 的 DCE 是拿**多态性**换来的——`when (T::class)` 是精确匹配：

- `T` 是 `Parcelable` 的**子类**（如 `MyBean::class`）→ 不等于 `Parcelable::class` → 走 `else`，返回 `false`（而 `set` 的 `is Parcelable` 能正确处理子类）
- `T = Number` 而运行时值是 `Int` → `set` 写成 Int 成功，`set2` 走 `else` 失败

所以如果你的 KV API 的所有调用点都传精确类型（通常如此），`set2` 更快且行为一致；如果需要支持父类型/子类型场景，就得用 `set` 的写法并放弃这份优化。
