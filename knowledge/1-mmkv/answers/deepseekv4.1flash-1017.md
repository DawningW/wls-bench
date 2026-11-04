## 结论

**`set`（第一个：`when (value)` + `is`）能内联并做死代码消除；`set2`（`when (T::class)`）不能，而且实际上更慢。**

我在本机用 Kotlin 2.1.0 + JDK 17 实际编译并用 `javap -c` 反编译验证过（`kttest/demo.kt`），不是纸上推演。

## 原因

- `is X` 在字节码里是 `instanceof`，是**类型检查**。内联后 `value` 的静态类型已被 reified 实参替换成具体类，编译器能静态判定分支真假，直接裁剪。
- `T::class == X::class` 是**运行时对象比较**，编译成 `Reflection.getOrCreateKotlinClass(...)` + `Intrinsics.areEqual(...)`，对编译器来说是不透明的函数调用，任何常量折叠/死代码消除都插不上手。

## 实测字节码（同一组 8 个分支）

| 调用点 | `set` | `set2` |
|---|---|---|
| `T = Int` | 1 次 `instanceof`（Boolean，恒假）+ 直接 `encode(String,int)`，61 字节 | 9 次 KClass 查询 + 8 次 `areEqual`，270 字节 |
| `T = Long` | 2 次 `instanceof` + 直接 `encode(String,long)`，92 字节 | 同上 |
| `T = String` | 5 次 `instanceof` + 直接 `encode(String,String)`，160 字节 | 同上，267 字节 |
| `T = User : Parcelable` | 8 次 `instanceof`，`else` 已被删掉 | 同上，276 字节 |

`set` 的特征很明确：**命中的那个分支变成无条件调用，它后面的所有分支连同 `else` 全部消失**；前面残留的几个 `instanceof` 编译器保守地没折（运行时永远为 false，JIT 类型特化后会剪掉，代价近乎为零）。

`set2` 则是原封不动的一条运行时比较链，8 个分支一个不少：

```
10: ldc class java/lang/String
12: invokestatic Reflection.getOrCreateKotlinClass   ← 每次调用都跑
19: getstatic  java/lang/Boolean.TYPE
22: invokestatic Reflection.getOrCreateKotlinClass
25: invokestatic Intrinsics.areEqual
28: ifeq 49      ... 重复 8 次
```

## 顺带一个更严重的问题：`set2` 是错的

`T::class` 只有 **T 恰好等于某个具体类** 时才匹配。实测（真实运行）：

```
set(User:Parcelable)   -> encode(Parcelable)   ✅
set(Any<-Int)          -> encode(Int)          ✅ 走 is，兜底正常
set2(User:Parcelable)  -> （什么都不发生）      ❌ 落到 else -> false
```

- `User : Parcelable` 时 `T::class == User::class`，不是 `Parcelable::class`，于是**静默丢弃这次写入**。`Parcelable::class` 这个分支实际上永远不可达。
- 泛型转发场景（`fun <T> put(v: T) = set2("k", v, 0)`，T 被擦成 `Any`）会 100% 落空。

所以这不只是性能问题，是语义 bug。建议直接删掉 `set2`，只留 `set`；如果想暴露写入失败，把 `.toUnit()` 换成返回 `Boolean` 即可（`set` 里 `else -> false` 在 `T` 具体时本来就被消除了，不留痕迹）。

需要的话我可以把 `kttest/demo.kt` 和 `dump.txt` 反编译结果留给你自己对照，或者顺手把 `set` 改成非 reified 版本（它其实不需要 `reified`，只有 `T::class` 才需要）。
