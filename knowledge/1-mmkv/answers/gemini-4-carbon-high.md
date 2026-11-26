# 结论

**`set`（第一个函数）能内联并进行死代码优化（DCE）；`set2`（第二个函数）虽然会被内联，但无法进行死代码优化（反而会导致调用处字节码膨胀，且存在严重的语义 Bug）。**

---

## 一、为什么 `set` 能内联并进行死代码优化？

```kotlin
actual inline fun <reified T> set(key: String, value: T, expire: Int) = when (value) {
    is Boolean -> mmkv.encode(key, value, expire)
    ...
    is Parcelable -> mmkv.encode(key, value, expire)
    else -> false
}.toUnit()
```

### 1. 编译与内联后的字节码形态
`set` 中的 `when (value)` 使用 `is` 类型判断，编译为标准的 JVM **`instanceof` + `checkcast`** 指令序列（注：`set` 函数体实际上并未使用 `reified` 特性，`<reified T>` 通常只是为了匹配 KMP `expect inline fun <reified T>` 签名；即使去掉 `reified`，仅靠 `inline` 效果也完全一致）。

当调用方传入具体类型（例如 `set("k", 123, 0)`、`set("k", "hello", 0)` 或 `set("k", userParcelable, 0)`）时：
1. **`kotlinc` 阶段**：完成函数体内联（注意：`kotlinc` 自身的字节码优化器主要折叠空检查与简单拆装箱，原始 `.class` 中通常仍保留 `instanceof` 分支，但内联已经把调用方实参的**精确静态类型与非空信息**暴露在了同一个方法体内）。
2. **R8 / D8 / ProGuard 及 ART AOT/JIT / JVM C2 阶段**：在单方法 SSA IR 上进行类型流分析（Type Lattice Propagation）与常量折叠：
   - **不兼容分支折叠为 `false`**：前 7 个分支的目标类型（`java.lang.Boolean`、`Integer`、`Long`、`Float`、`Double`、`String`、`byte[]`）在 JVM 上全部是 **`final` 类**，且均未实现 `Parcelable`。当实参静态类型已知（如 `String`、`Integer.valueOf(...)` 返回的 `Integer`、或具体的 `User : Parcelable`）时，与其他所有不兼容类型的 `instanceof` 在静态上恒为 `false`（即便不依赖非空分析也恒为 `false`）。
   - **目标分支折叠为 `true`**：对于已知非空的实参（字面量、构造器对象、Kotlin 非空类型引用，以及 R8/JIT 建模为非空的 `Integer.valueOf` 返回值），匹配自身类型或父接口（如 `User instanceof Parcelable`）的 `instanceof` 恒为 `true`。
   - **死代码消除（DCE）与标量替换/拆箱**：所有恒假分支及后续不可达的 `else` 分支被 **Dead Code Elimination** 彻底移除，多余的 `checkcast` 被消去；对于基本类型（如 `Int`），`Integer.valueOf(123)` 失去了逃逸引用，与 `intValue()` 配对后被 R8 / ART / JIT 直接拆箱消除。

最终，`set("k", 123, 0)` 在优化后会被直接化简为单条调用：
```java
mmkv.encode("k", 123, 0);
```
即使实参是可空的具体类型（如 `val s: String?`），其余 7 个无关的 `encode` 分支依然会被全部作为死代码消除，仅退化为一次判空后调用 `mmkv.encode(key, s, expire)`。

---

## 二、为什么 `set2` 无法进行死代码优化？

```kotlin
actual inline fun <reified T> set2(key: String, value: T, expire: Int) = when (T::class) {
    Boolean::class -> mmkv.encode(key, value as Boolean, expire)
    ...
    Parcelable::class -> mmkv.encode(key, value as Parcelable, expire)
    else -> false
}.toUnit()
```

### 1. 直觉误区：`reified` + `T::class` 并不是编译期 `if constexpr`
很多人直觉上认为 `T` 有 `reified` 修饰，`T::class` 在编译期已知，因此 `when (T::class)` 会在编译期常量折叠。**但在 Kotlin 中并非如此**：`T::class` 的类型是 `kotlin.reflect.KClass<T>`，而不是编译期常量原语。

以 `set2("k", 123, 0)`（`T = Int`）为例，内联后生成的等价逻辑是：
```java
Object value = Integer.valueOf(123);
KClass var4 = Reflection.getOrCreateKotlinClass(Integer.TYPE);
if (Intrinsics.areEqual(var4, Reflection.getOrCreateKotlinClass(Boolean.TYPE))) {
    mmkv.encode(key, ((Boolean) value).booleanValue(), expire);
} else if (Intrinsics.areEqual(var4, Reflection.getOrCreateKotlinClass(Integer.TYPE))) {
    mmkv.encode(key, ((Integer) value).intValue(), expire);
} else if (Intrinsics.areEqual(var4, Reflection.getOrCreateKotlinClass(Long.TYPE))) {
    ...
} ...
```

### 2. 阻碍静态分析与 DCE 的根本原因
无论是 `kotlinc` 还是 R8 / D8，都**无法**将上述 `when (T::class)` 条件在编译期折叠为布尔常量：
1. **动态反射工厂初始化**：`Reflection.getOrCreateKotlinClass(...)` 内部委托给 `Reflection.factory`。而 `Reflection.<clinit>` 通过 `Class.forName("kotlin.reflect.jvm.internal.ReflectionFactoryImpl")` 动态加载反射实现（无 `kotlin-reflect` 时回退到 `new ClassReference(jClass)`），属于带静态初始化副作用的多态调用。
2. **复杂的运行时等价比较**：`when` 分支使用 `Intrinsics.areEqual(a, b)` 调用 `ClassReference.equals`。该方法内部调用 `JvmClassMappingKt.getJavaObjectType`，在运行时通过 Native 方法 `Class.isPrimitive()` 以及 `Class.getName()` 字符串匹配（`"int"`、`"boolean"` 等）将基本类型 `Class` 映射为包装类 `Class` 后再比较。
3. **内联反而引发负优化**：由于优化器无法静态证明这些 `Intrinsics.areEqual` 的返回值与无副作用性，每次调用 `set2` 都会在调用点内联展开多达 **8 次 `Reflection.getOrCreateKotlinClass` + `Intrinsics.areEqual` 调用链**，不仅无法消除死分支、导致严重的字节码膨胀，还保留了运行期创建/查找 `KClass` 对象的开销以及基本类型参数 `value` 的装箱开销。

> **补充：如果把 `T::class` 改成 `T::class.java` 呢？**
> 改成 `T::class.java` 虽然在字节码层面变为 `Class` 常量（R8 可以折叠非基本类型的 `Intrinsics.areEqual(Class, Class)`），但立刻会踩中 Kotlin 基本类型与可空包装类型的 `Class` 不一致坑（`Int::class.java` 是 `int.class`，而 `(Int?)::class.java` 是 `Integer.class`），且依然存在下述 `Parcelable` 子类匹配失败等严重 Bug。

---

## 三、除此之外，`set2` 还存在三个严重的语义 Bug

即便抛开性能优化不谈，`set2` 在功能语义上也是错误的：

1. **`Parcelable` 子类永远写入失败**：
   `Parcelable` 是接口。实际调用 `set2("user", user, 0)` 时，`T` 被推导为具体实现类（如 `User`），`T::class` 是 `User::class`。`when (T::class)` 执行的是**精确类相等（`==`）**而非子类型判断（`is`），`User::class == Parcelable::class` 恒为 **`false`**，会直接落入 `else -> false` 导致所有自定义 `Parcelable` 对象静默保存失败。
2. **可空类型（`T?`）传 `null` 会抛 `NullPointerException`**：
   例如 `val v: String? = null; set2("k", v, 0)`，`T` 为 `String?`，但 `(String?)::class` 依然是 `String::class`，会命中 `String::class` 分支并执行 `value as String`，触发 `NullPointerException`（而 `set` 中 `null is String` 为 `false`，会安全落入 `else -> false`）。
3. **静态类型被擦除/向上转型为 `Any` 时失效**：
   例如 `val v: Any = "hello"; set2("k", v, 0)`，`T` 为 `Any`，`Any::class` 无法匹配任何分支而返回 `false`；而 `set` 检查的是运行期实参 `value` 的真实类型，仍能正确编码。

---

## 四、总结对比

| 维度 | `set` (`when (value) { is ... }`) | `set2` (`when (T::class) { ...::class -> }`) |
| :--- | :--- | :--- |
| **内联支持** | 支持 | 支持 |
| **死代码优化 (DCE)** | **能**（R8/D8 及 ART/JIT 基于 SSA 类型分析折叠 `instanceof`、消除不可达分支并拆箱） | **不能**（`Reflection.getOrCreateKotlinClass` + `Intrinsics.areEqual` 阻断编译期常量折叠） |
| **基本类型装箱消除** | **能**（DCE 后仅剩单分支拆箱，可被标量替换消除装箱） | **不能**（`value` 作为 `Object` 保留且多分支存活） |
| **代码体积影响** | 优化后收敛为单条 `mmkv.encode(...)` 调用 | 每个调用点膨胀出 8 组反射 `KClass` 获取与判等代码 |
| **`Parcelable` 子类支持** | **正确**（`instanceof Parcelable` 支持子类型） | **错误**（`User::class != Parcelable::class`，永远走 `else`） |
| **可空类型 `null` 安全性** | **安全**（`null is X` 为 `false`，走 `else`） | **崩溃**（`T?::class` 仍命中 `T::class`，`value as X` 抛 NPE） |

---

> 以下为第二轮独立复测（Round 2）原始回答：

# 结论

**`set`（第一个函数）能够内联并进行死代码优化（Dead Code Elimination, DCE）**；而 **`set2`（第二个函数）虽然能内联，但无法进行死代码消除**，并且还会引发代码膨胀、运行时反射开销以及严重的语义 Bug。

---

# 原理深度解析

## 1. 为什么 `set` 能内联并完成死代码优化（DCE）？

```kotlin
actual inline fun <reified T> set(key: String, value: T, expire: Int) = when (value) {
    is Boolean -> mmkv.encode(key, value, expire)
    is Int -> mmkv.encode(key, value, expire)
    ...
    is Parcelable -> mmkv.encode(key, value, expire)
    else -> false
}.toUnit()
```

### 编译与优化链路：
1. **字节码生成与内联（`kotlinc` 阶段）**：
   - `when (value) { is ... }` 会编译为标准的 JVM 字节码类型判断与转换指令：`instanceof` + `checkcast`（基本类型伴随自动装拆箱，如 `Integer.valueOf` / `Number.intValue`）。
   - 虽然 `kotlinc` 自身的字节码内联器在生成 `.class` 时只是将函数体机械内联到调用处（尚未裁剪分支），但内联使得**调用处实参 `value` 的精确静态类型暴露在了同一个方法体内**。
2. **静态类型传播与死代码消除（Android R8 / ProGuard 及 ART AOT / JVM C2 JIT 阶段）**：
   - 以调用 `set("k", 123, 0)` 为例，内联后局部变量 `value` 的 SSA 静态类型被精确推导为非空的 `java.lang.Integer`。
   - 前 7 个分支的目标类型（`Boolean`、`Integer`、`Long`、`Float`、`Double`、`String`、`byte[]`）在 JVM/Android 中全部是 **`final` 类型**，彼此完全互斥，且均未实现 `Parcelable` 接口。
   - 优化器（如 R8 的 `InstanceOf` 常量折叠 + `DeadCodeRemover`）在编译期即可静态判定：
     - `value instanceof Boolean` 恒为 `false` $\rightarrow$ 分支消除；
     - `value instanceof Integer` 恒为 `true` $\rightarrow$ 条件判断消除；
     - 后续 `Long`、`Float`、`Double`、`String`、`byte[]`、`Parcelable` 及 `else -> false` 全部不可达 $\rightarrow$ **彻底被死代码消除（DCE）**；
     - 同时触发拆箱消除（Unboxing Elimination），将 `Integer.valueOf(123).intValue()` 直接还原为基础类型 `123`，消除装箱对象分配。
   - **最终产物**：整个 `set("k", 123, 0)` 在内联与 DCE 后直接退化为单条零开销调用：
     ```java
     mmkv.encode("k", 123, 0);
     ```
   - **其他类型表现**：
     - 传入具体 `Parcelable` 实现类（如 `User`）甚至仅静态声明为非空 `Parcelable` 接口时，由于前 7 个类型均为不实现 `Parcelable` 的 `final` 类，前 7 次 `instanceof` 同样恒为 `false`，`value instanceof Parcelable` 恒为 `true`，同样能完成完整 DCE。
     - 即便传入可空具体类型（如 `Int?`），也能消除其余 7 个无关类型分支，仅保留一次判空与对应的 `encode` 调用。

> **补充说明**：在 `set` 中真正使内联后 DCE 生效的是 **`inline` + 基于值的 `is`（`instanceof`）指令**，其实并不依赖 `reified`（此处保留 `reified` 通常只是为了匹配 KMP 中 `expect inline fun <reified T>` 的函数签名）。

---

## 2. 为什么 `set2` 无法进行死代码优化？

```kotlin
actual inline fun <reified T> set2(key: String, value: T, expire: Int) = when (T::class) {
    Boolean::class -> mmkv.encode(key, value as Boolean, expire)
    Int::class -> mmkv.encode(key, value as Int, expire)
    ...
    Parcelable::class -> mmkv.encode(key, value as Parcelable, expire)
    else -> false
}.toUnit()
```

开发者常误以为“`T` 是 `reified`，因此 `when (T::class)` 会在编译期求值并裁剪分支”，但实际生成的字节码完全不是编译期常量比较：

1. **`T::class` 生成的是运行时 `KClass` 包装调用**：
   - 内联替换实化类型参数 `T`（如 `T = Int`）后，`T::class` 会编译为：
     `Reflection.getOrCreateKotlinClass(Integer.class)`。
   - 每个分支条件则会编译为：
     `Intrinsics.areEqual(subject, Reflection.getOrCreateKotlinClass(Boolean.TYPE))` 等方法调用。
2. **优化器（`kotlinc` / R8 / ProGuard）无法常量折叠 `KClass` 相等性判断**：
   - **静态初始化反射副作用**：`Reflection.getOrCreateKotlinClass` 依赖静态字段 `Reflection.factory`，而 `Reflection.<clinit>` 中包含 `Class.forName("kotlin.reflect.jvm.internal.ReflectionFactoryImpl")` 的反射加载逻辑（存在类初始化副作用；若引入了 `kotlin-reflect` 还会走全局并发缓存与元数据解析）。
   - **运行时对象分配与复杂的类型映射逻辑**：在未引入 `kotlin-reflect` 的常规 Android 环境下，`getOrCreateKotlinClass` 每次都会创建 `new ClassReference(jClass)` 实例。更关键的是，实化参数 `T::class`（当 `T = Int` 时）底层持有的是包装类 `java.lang.Integer.class`，而分支字面量 `Int::class` 底层持有的是基本类型 `int.class`（`Integer.TYPE`）。两者之所以在运行时相等，是因为 `ClassReference.equals()` 调用了 `JvmClassMappingKt.getJavaObjectType()`，内部通过 `Class.isPrimitive()`（native 方法）以及对 `Class.getName()` 字符串做多路哈希 `switch` 来将基本类型归一化为包装类。
   - 这种跨方法的堆分配、`<clinit>` 副作用、`Class` native 方法调用以及字符串哈希分发，超出了 `kotlinc` 与 R8/ProGuard 的编译期常量折叠能力。
3. **最终后果（内联膨胀而非优化）**：
   - `set2` 被内联到每个调用点后，**所有 8 个分支的 `Reflection.getOrCreateKotlinClass` + `Intrinsics.areEqual` 判断逻辑都会原样保留**，造成严重的字节码体积膨胀；
   - 基本类型实参被迫保留装箱（`Integer.valueOf`），且每次调用都会在运行时重复分配多个 `ClassReference` 对象并逐一比对。

---

## 3. 附：`set2` 存在的严重语义 Bug

除了无法进行死代码优化外，`set2` 与 `set` 在语义上也并不等价，存在多处运行时缺陷：

1. **`Parcelable` 实现类全部存储失败**：
   `Parcelable` 是接口，实际调用如 `set2("user", user, 0)`（其中 `class User : Parcelable`）时，实化类型 `T` 被推导为具体的 `User`。由于 `User::class == Parcelable::class` 为 `false`，会直接落入 `else -> false` 导致所有具体 `Parcelable` 对象静默写入失败；而 `set` 中的 `value is Parcelable` 能正确匹配任意实现类。
2. **可空类型触发 `NullPointerException`**：
   当传入可空变量且值为 `null` 时（例如 `val s: String? = null; set2("k", s, 0)`），Kotlin 中 `(String?)::class` 依然等于 `String::class`，会命中分支并执行 `value as String`，直接抛出 `NullPointerException`；而 `set` 中 `null is String` 为 `false`，会安全走 `else -> false`。
3. **静态类型为父类型时失效**：
   若变量静态类型为 `Any`（如 `val v: Any = "hello"`），`set2` 中 `T::class` 为 `Any::class`，无法匹配任何具体分支；而 `set` 仍可根据运行时实例类型正确分发。

---

# 对比总结

| 维度 | `set` (`when (value) { is ... }`) | `set2` (`when (T::class) { ... }`) |
| :--- | :--- | :--- |
| **底层指令** | 标准 `instanceof` + `checkcast` | `Reflection.getOrCreateKotlinClass` + `Intrinsics.areEqual` |
| **能否内联** | 能 | 能（反而导致调用点代码膨胀） |
| **死代码消除 (DCE)** | **能**（R8 / ART / JIT 可完全消除无关分支及装箱，退化为单次 `encode` 直调） | **不能**（含反射工厂初始化、`ClassReference` 分配及 `Class.getName()` 归一化，无法静态折叠） |
| **`Parcelable` 子类支持** | 正常（`is` 支持子类型判断） | **Bug**（`SubClass::class != Parcelable::class`，全部走 `else`） |
| **可空类型 `null` 安全性** | 安全（`null is X` 为 `false`） | **Bug**（`(X?)::class == X::class`，强转 `value as X` 抛 NPE） |
