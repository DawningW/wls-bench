两个函数都会内联，但仅凭 Kotlin 的 inline reified，不能保证任何一个在 JVM 编译阶段完成死代码消除。

  关键区别是：

  - set 判断的是 value 的运行时类型，语义正确且支持 Parcelable 的实现类。
  - set2 判断的是调用点传入的泛型类型 T，而且是精确的 KClass 相等，不是子类型判断。

  例如：

  set<Any>("key", 1, expire)   // 命中 is Int
  set2<Any>("key", 1, expire)  // T::class 是 Any::class，返回 false

  set("key", myParcelable, expire)   // 命中 is Parcelable
  set2("key", myParcelable, expire)  // T 是 MyParcelable，不等于 Parcelable，返回 false

  关于优化：

  1. 两者都会在调用点展开，因为都是 inline。
  2. reified 会让 T::class 在调用点具体化，所以理论上 set2 更容易被后端优化器分析。
  3. 但 JVM 上的 Kotlin 内联本质上主要是字节码展开，不会保证重新进行完整的类型分析和常量折叠；T::class == Xxx::class 的分支通常仍可能保留。
  4. 最终是否消除无效分支取决于 Kotlin 编译器版本、目标后端，以及 R8/ProGuard 等后续优化器。

  所以准确结论是：

  > 两个函数都能内联；两个都可能被后端优化，但 Kotlin 语言层面都不保证死代码消除。set2 看起来更适合静态优化，却与 set 语义不同，并且其 Parcelable::class 分支通常无法处理具体实现
  > 类，因此不应为了假定的优化而改用 set2。

  如果要确认特定 Android/Kotlin 版本的实际结果，必须检查 release 构建经过 R8 后的字节码或 DEX，而不能仅根据源码判断。

• 我会把“语言层保证”“Kotlin/JVM 编译产物”和“JIT/R8 可能做的优化”分开说；这三层混在一起，很容易得到错误结论。

• Ran command -v kotlinc && kotlinc -version
  └ (no output)

───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

• 结论很明确：

  - set 和 set2 都会内联，因为它们都有 inline。
  - 仅靠 Kotlin/JVM 编译器，两个函数都不保证在编译期消除其他 when 分支。
  - set2 理论上更容易被后续优化器做死代码消除，因为内联后 T 已确定；但 reified 只保证具体化类型信息，不保证常量折叠或 DCE。
  - Android 的 R8、ART JIT/AOT 可能进一步消除分支，但这是优化器行为，不能作为语言层保证。

  内联 set2<Int>() 后，代码在概念上变成：

  when (Int::class) {
      Boolean::class -> ...
      Int::class -> ...
      Long::class -> ...
  }

  但 Kotlin/JVM 生成的字节码通常仍包含一串 KClass 比较。Kotlin 内联器不会承诺直接把它化简成：

  mmkv.encode(key, value as Int, expire)

  set 内联后依然基于运行时对象做 instanceof 判断：

  when (value) {
      is Boolean -> ...
      is Int -> ...
  }

  如果优化器能证明 value 一定是 Int，它也可能消除其他分支，但同样不保证。

  更重要的是，两个函数语义不同，set2 有实际错误：

  class User : Parcelable

  set2("user", User(), 0)

  这里推导出的 T 是 User：

  T::class == User::class
  Parcelable::class == Parcelable::class

  因此不会命中 Parcelable::class 分支，最终返回 false。而 set 的：

  value is Parcelable

  可以正确命中。

  同样：

  val value: Any = 123

  set("k", value, 0)  // 命中 Int
  set2("k", value, 0) // T 是 Any，落入 else

  所以最终判断是：

  > 两者都能内联；两者都没有 Kotlin 层保证的死代码优化。set2 更具备静态消除分支的条件，但依赖 R8/JIT/AOT，而且语义不等价，在 Parcelable 子类和宽静态类型场景下会出错。实际代码应选
  > 择 set，不要为了未经保证的 DCE 改成 set2。
