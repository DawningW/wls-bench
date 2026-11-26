# 1-mmkv：哪个函数能内联并进行死代码优化？

## 结论

**两个函数都会被内联**（`inline` + `reified`，kotlinc 一定会把函数体复制到调用处），
**真正有机会做死代码消除（DCE）的是第一个 `set`（`when (value) { is X -> ... }`）**。
第二个 `set2`（`when (T::class)`）虽然看起来像“编译期已知类型”，实际上**无法被消除**，
所有分支都会保留，并且每次调用都要在运行时做 KClass 的创建和比较。

---

## 为什么 `set2` 做不了 DCE

`reified T` 的作用只是：在内联时，把字节码里 `T` 的占位符换成具体类。它**不会**在编译期计算 `when` 的条件。

`T::class` 和 `Int::class` 这种类字面量，在 JVM 字节码里是**方法调用**：

```java
KClass t = Reflection.getOrCreateKotlinClass(Integer.class);   // T::class，T = Int
if (Intrinsics.areEqual(t, Reflection.getOrCreateKotlinClass(Boolean.TYPE))) { ... }
else if (Intrinsics.areEqual(t, Reflection.getOrCreateKotlinClass(Integer.TYPE))) { ... }
else if ...
```

- `when(subject)` 的分支比较用的是 `equals`（`Intrinsics.areEqual`），两边都是运行时创建的 `KClass` 对象（`ClassReference`/`KClassImpl`）。
- kotlinc 不知道 `Reflection.getOrCreateKotlinClass` 和 `KClass.equals` 的语义，所以无法把这些条件当成常量；R8/ProGuard 也只把它们当普通方法调用，同样没法折叠。
- 结果就是：每次调用 `set2` 都会把**所有分支的字节码**带进调用处（代码膨胀），运行时还要逐个创建 KClass 并调用 `equals` 依次比较，比 `instanceof` 慢得多。

## 为什么 `set` 可以 DCE

内联之后，`when (value)` 里的每个分支只是一条 `INSTANCEOF` 指令加一个分支跳转，比较对象就是实参本身：

1. **基本类型（Boolean/Int/Long/Float/Double）**：泛型参数 `T` 会让实参先装箱（例如 `Integer.valueOf(1)`），再传进内联代码。kotlinc 的 JVM 后端在内联之后会跑一组方法级优化：
   - 冗余装箱消除（RedundantBoxing）：知道这个装箱值的确切类型是 `Integer`，所以能把 `INSTANCEOF Boolean` 换成常量 `false`、`INSTANCEOF Integer` 换成 `true`，并把 `checkcast + intValue()` 拆箱这一对操作抵消掉；
   - 常量条件消除 + 死代码消除：去掉永远走不到的分支。

   最终调用处可能只剩一行 `mmkv.encode(key, 1, expire)`，既没有装箱，也没有多余分支。
2. **引用类型（String/ByteArray/Parcelable）**：kotlinc 自己不对引用类型做类型传播，`instanceof` 链会留在字节码里。但这些是标准的 `instanceof`，静态类型已知，**R8（Android release 构建）的类型分析可以折叠 `instanceof` 并删掉死分支**，JIT（ART/C2）在运行时也能做同样的优化。
3. 就算没有被优化掉，`instanceof` 链本身也很便宜，不分配对象。

---

## 两者的语义差异（`set2` 还有正确性问题）

| 场景 | `set`（`is`） | `set2`（`T::class`） |
|---|---|---|
| `Parcelable` 的实现类（如 `User : Parcelable`） | `is Parcelable` 能匹配 ✅ | `T::class == User::class`，**不等于** `Parcelable::class`，掉进 `else -> false`，**静默不保存** ❌ |
| 静态类型更宽（`val v: Any = 1; set2(k, v, e)`） | 按运行时类型匹配到 Int ✅ | `T = Any`，掉进 `else`，静默不保存 ❌ |
| 可空类型（`T = Int?`，值为 null） | 所有 `is` 都不成立 → `false`，安全 | 可能匹配到 `Int::class`，然后 `null as Int` 抛 NPE（或类型转换异常） ❌ |
| `Int::class` 和 `T::class` 的比较 | — | `Integer.TYPE` 和 `Integer.class` 的 KClass 按 `javaObjectType` 比较会相等，能工作，但只能在运行时判断 |

另外，`set` 里的 `reified` 其实没有用到（`is` 检查的都是具体类型，不是 `T`）。去掉 `reified` 甚至 `inline`，语义也不变。保留 `inline` 的唯一好处就是上面说的：调用处能拿到实参的具体类型，从而让装箱消除和 DCE 生效。

## 一句话总结

> 选 **`set`（`when (value)` + `is`）**：内联后变成 `instanceof` 链，基本类型实参可以被 kotlinc 消除装箱、折叠常量、删掉死分支，引用类型可以交给 R8/JIT 折叠；语义上还支持子类和多态。
> `set2` 的 `T::class` 比较在字节码里是 `Reflection.getOrCreateKotlinClass(...)` + `equals` 这样的运行时调用，编译器和 R8 都没法把它当常量，**做不了死代码消除**。它还有 Parcelable 子类和宽类型静默丢数据、可空值可能抛 NPE 的问题。
