# 1-mmkv 静态分析结论

## 结论

- **内联能力**：`set` 和 `set2` 均声明为 `inline`，因此在调用处**都可以被 Kotlin 编译器（`kotlinc`）内联**。
- **死代码优化（DCE）能力**：**`set` 可以进行死代码优化，而 `set2` 不能。**
  - 更精确地说：在 `kotlinc` 自身的 JVM 字节码内联阶段，两个函数的 `when` 分支都会被完整复制到调用点；但在后续的 **Android R8 / ProGuard（以及 ART / JVM JIT）优化阶段**，**`set` 能够触发有效的死代码消除（Dead Code Elimination）和装箱消除**，而 **`set2` 无法被静态优化**，内联后反而会在每个调用点留下全部冗余分支和反射相关调用，导致代码膨胀与运行时性能下降。

---

## 详细原因分析

### 1. 为什么 `set` 可以内联并被死代码优化

```kotlin
actual inline fun <reified T> set(key: String, value: T, expire: Int) = when (value) {
    is Boolean -> mmkv.encode(key, value, expire)
    ...
    is Parcelable -> mmkv.encode(key, value, expire)
    else -> false
}.toUnit()
```

1. **编译为原生 `INSTANCEOF` 指令**  
   `when (value)` 中的 `is Boolean`、`is Int` 等分支在 JVM 字节码中直接编译为标准的 `INSTANCEOF` 指令链（例如 `instanceof java/lang/Boolean`、`instanceof java/lang/Integer`、`instanceof [B`、`instanceof android/os/Parcelable` 等）。
2. **调用点内联保留了具体类型信息**  
   尽管 `set` 的参数 `value: T` 在字节码层面擦除为 `java.lang.Object`（基本类型实参在进入内联块前会先调用 `Integer.valueOf(...)` 等装箱），但内联之后，调用点的局部变量或表达式类型对字节码优化器是完全可见的：
   - 例如 `set("k", 123, 0)` 中，`value` 来自 `Integer.valueOf(123)`，R8 可精确推导其为**非空且精确类型为 `java.lang.Integer`**。
   - 例如 `set("k", str, 0)` 中，`value` 的静态类型为 `java.lang.String`。
3. **R8 / JIT 可静态折叠并消除死代码**  
   - `Boolean`、`Integer`、`Long`、`Float`、`Double`、`String`、`byte[]`（`ByteArray`）在 JVM 上全部是 **`final` 类型（或数组类型）**，彼此之间不可继承，也不实现 `Parcelable`。
   - 因此，当调用处传入上述任一具体类型时，R8 的静态类型分析可以直接确定：
     - 所有不匹配的 `instanceof` 检查恒为 `false`（分支被剪除）；
     - 匹配的 `instanceof` 检查（若已知非空）恒为 `true`（检查被移除，直接进入对应的 `mmkv.encode` 调用）；
     - 其余后续分支与 `else -> false` 成为不可达代码，被 **DCE 彻底删除**；
     - 对于基本类型（如 `Int`），R8/JIT 还可以进一步消除 `Integer.valueOf(x)` 与 `.intValue()` 这对冗余的装箱/拆箱操作，最终接近直接调用 `mmkv.encode(key, x, expire)`。
   - 当传入某个具体的 `Parcelable` 实现类（如 `User`）时，前面 7 个与 `User` 互斥的分支也都会被静态消除；若非空，`is Parcelable` 检查和 `else` 也会被消除。只有在调用点静态类型本身就是 `Any` 等宽泛父类型时，才会保留运行时 `instanceof` 判断（这也是保证多态分发正确性的必要开销）。

---

### 2. 为什么 `set2` 能内联但无法进行死代码优化

```kotlin
actual inline fun <reified T> set2(key: String, value: T, expire: Int) = when (T::class) {
    Boolean::class -> mmkv.encode(key, value as Boolean, expire)
    ...
    Parcelable::class -> mmkv.encode(key, value as Parcelable, expire)
    else -> false
}.toUnit()
```

1. **`when (T::class)` 不会在 `kotlinc` 阶段被常量折叠**  
   虽然 `T` 是 `reified` 类型参数，`kotlinc` 内联 `set2` 时会将 `T::class` 中的 `T` 替换为调用点实参类型（例如 `Integer.TYPE` 或 `String::class.java`），但 JVM 字节码并没有原生的 `KClass` 常量池项，`kotlinc` 也不会在编译/内联期把 `when (T::class)` 折叠为单一分支。
2. **字节码退化为大量反射包装调用与 `equals` 比较**  
   在 JVM 字节码中：
   - `T::class` 以及 `Boolean::class`、`Int::class` 等 8 个分支条件都会被编译为 `Reflection.getOrCreateKotlinClass(Class)` 调用（共 9 次）；
   - `when` 的每个条件分支都会被编译为 `Intrinsics.areEqual(kclassA, kclassB)` 调用（内部判空并执行虚方法 `KClass.equals(Object)`）。
3. **优化器（R8 / ProGuard / JIT）无法消除这些分支**  
   - `Reflection.getOrCreateKotlinClass` 通过静态字段 `Reflection.factory` 分发（其 `<clinit>` 中通过 `Class.forName("kotlin.reflect.jvm.internal.ReflectionFactoryImpl")` 动态探测 `kotlin-reflect`；无 `kotlin-reflect` 时每次调用都会实例化 `new ClassReference(javaClass)`，有 `kotlin-reflect` 时则查询内部缓存）。
   - 因为涉及动态工厂、对象分配/缓存查询以及虚方法 `equals` 调用，R8 无法将这些调用视为编译期纯常量，也无法静态证明 `Intrinsics.areEqual(getOrCreateKotlinClass(A), getOrCreateKotlinClass(B))` 的真假。
   - 结果是：**所有 9 次 `getOrCreateKotlinClass` 调用、8 次 `Intrinsics.areEqual` 判断、类型强转以及全部 9 个分支都会完整保留在每个内联调用点**，完全无法进行死代码消除，反而比普通非内联函数带来更大的字节码膨胀和额外对象开销。

---

## 补充：`set2` 存在的语义缺陷

除了无法进行死代码优化外，`set2` 在行为上还存在两处明显 Bug，而 `set` 均能正确处理：

1. **破坏子类型与多态（尤其是 `Parcelable` 子类）**  
   `when (T::class)` 比较的是精确的 `KClass` 相等，而非子类型关系。调用 `set2("user", myParcelable, 0)` 时，推导出的 `T` 是具体子类（如 `User`），`User::class != Parcelable::class`，因此会跳过 `Parcelable` 分支直接落入 `else -> false`（除非调用方手动写死 `set2<Parcelable>(...)`）；若入参静态类型为 `Any`，`set2` 同样会全部失败。`set` 使用 `value is Parcelable` 则天然支持多态和子类型。
2. **可空类型传入 `null` 时崩溃**  
   若以可空类型调用 `set2<String?>("k", null, 0)` 或 `set2<Int?>("k", null, 0)`，`T::class` 仍会匹配 `String::class` / `Int::class` 分支（`ClassReference.equals` 比较 `javaObjectType`，因此 `Integer.class` 与 `Integer.TYPE` 对应的 `KClass` 也相等），随后执行非空强转 `value as String` / `value as Int` 会直接抛出 `NullPointerException`。而 `set` 中 `null is String` / `null is Int` 结果为 `false`，会安全走进 `else -> false`。

---

> 以下为第二轮独立复测（Round 2）原始回答：

# 题目（1-mmkv）解答

## 结论

- **内联能力**：`set` 和 `set2` 都声明了 `inline`，在 Kotlin 编译阶段**都可以被内联**。
- **死代码优化（Dead Code Elimination, DCE）**：**`set` 可以进行死代码优化，`set2` 不能。**

---

## 详细分析

### 1. 为什么 `set` 能内联并触发死代码优化？

`set` 的分支条件是针对参数 `value` 的 `is` 类型检查：

```kotlin
when (value) {
    is Boolean -> mmkv.encode(key, value, expire)
    is Int -> mmkv.encode(key, value, expire)
    ...
    is Parcelable -> mmkv.encode(key, value, expire)
    else -> false
}
```

1. **内联展开**：调用 `set(...)` 时，`kotlinc` 会把函数体直接内联到调用点，实参的静态/已知具体类型会直接暴露在内联后的局部控制流中（注：`kotlinc` 的标准 JVM 字节码内联本身不会直接删除这些 `when` 分支，但会生成供后续优化器处理的规范指令）。
2. **原生 `INSTANCEOF` 指令**：`is Type` 在 JVM / Dalvik 字节码中编译为原生的 **`INSTANCEOF`（`instance-of`）** 指令，这是 **R8（Android 发布优化器）、ART（`dex2oat`）以及 JVM JIT（HotSpot C1/C2）** 最擅长进行类型传播（Type Propagation）与常值折叠的指令。
3. **死分支消除**：
   - `Boolean`、`Integer`、`Long`、`Float`、`Double`、`String` 和 `ByteArray`（`[B`）在 JVM 中均为**互不相交的 `final` 类型**，且它们都不实现 `android.os.Parcelable`。
   - 当在调用点传入例如 `String` 字面量/`String` 变量，或传入 `Int`（调用点虽因泛型 `T` 擦除为 `Object` 而先执行 `Integer.valueOf(x)` 装箱，但 `Integer.valueOf` 返回值已知为非空 `java.lang.Integer`），或传入具体的 `Parcelable` 实现类时：
     - **R8**（以及 ART / HotSpot JIT）可以根据已知类型在编译期/优化期静态判定：所有不匹配分支的 `INSTANCEOF` **恒为 `false`**，从而将这些死分支连同里面的 `CHECKCAST`、拆箱和 `mmkv.encode(...)` 调用全部作为死代码消除。
     - 当实参类型同时已知非空（如字面量 `"abc"` 或 `Integer.valueOf(123)`）时，匹配分支的 `INSTANCEOF` 还会被静态折叠为 `true`，连冗余的 `CHECKCAST` 和 `else -> false` 分支也会一并被消除。
   - 顺带一提：`set` 函数体中实际上并未使用 `reified T`，它的死代码优化依赖的是 **`inline` 内联暴露实参类型 + `INSTANCEOF`**，而不是 `reified`。

---

### 2. 为什么 `set2` 能内联但无法进行死代码优化？

`set2` 表面上利用了 `reified T` 写出了 `when (T::class)`：

```kotlin
when (T::class) {
    Boolean::class -> mmkv.encode(key, value as Boolean, expire)
    Int::class -> mmkv.encode(key, value as Int, expire)
    ...
}
```

虽然内联时 `kotlinc` 会把 `T::class` 中的 `T` 替换为具体调用点的类型，但 **`when (KClass)` 在 Kotlin/JVM 上并不会被编译期常量折叠，也不会生成 `INSTANCEOF`**：

1. **复杂的反射包装与虚方法比较**：
   - `T::class` 与各个分支的 `Boolean::class`、`Int::class` 等，在 JVM 字节码中都会被编译为 `Reflection.getOrCreateKotlinClass(Class)`，返回 `kotlin.reflect.KClass` 对象（未引入 `kotlin-reflect` 时每次构造 `ClassReference`，引入时经由 `ReflectionFactoryImpl` 查找/创建 `KClassImpl`）。
   - `when (T::class)` 会编译为逐分支调用 `Intrinsics.areEqual(kclass1, kclass2)`，其底层会进行虚方法 `KClass.equals` 分发。
   - 在 `ClassReference.equals` 内部，为了使基本类型 `int.class`（`Integer.TYPE`）与包装类 `Integer.class` 相等，还会通过 `javaObjectType` 调用 `Class.isPrimitive()` 以及对 `Class.getName()` 做字符串分支映射。
2. **编译器与 R8 均无法静态消除**：
   - `kotlinc` 不会对 `Reflection.getOrCreateKotlinClass(...)` 与 `Intrinsics.areEqual(...)` 做编译期常量折叠。
   - `Reflection` 的静态初始化块中使用了 `Class.forName("kotlin.reflect.jvm.internal.ReflectionFactoryImpl")`，加上虚方法 `equals` 和 `Class.isPrimitive()` / `Class.getName()`，**R8 无法在编译期静态求值这些 `KClass` 比较结果**。
   - 因此，**所有 8 个条件判断和分支骨架在每个内联调用点都会被完整保留，无法被死代码消除**（即便某些死分支内部的 `value as OtherType` 能被 R8 推断出必然抛出 `ClassCastException`，外层的 `Intrinsics.areEqual` 条件判断与反射调用本身依然无法被消除），反而会导致明显的字节码膨胀和额外的运行时反射/分配开销。

---

### 3. 补充：`set2` 除无法优化外还存在的语义缺陷

1. **多态与 `Parcelable` 子类失效**：
   `when (T::class)` 判断的是**精确类相等（`==`）**而非子类型关系（`is`）。实际调用 `set2("u", user, 0)`（其中 `User : Parcelable`）时，`T` 被推断为 `User`，而 `User::class != Parcelable::class`，会直接落入 `else -> false` 导致静默写入失败；当参数静态类型为 `Any` / `Number` / `CharSequence` 时同理全部失效。`set` 使用 `is` 运行时类型判断则完全正确。
2. **可空类型传入 `null` 时崩溃**：
   当 `T` 为可空类型（如 `Int?`、`String?`）且 `value == null` 时，`T::class` 依然是 `Int::class` / `String::class`，`set2` 会匹配对应分支并执行 `value as Int` / `value as String` 强转，导致运行时抛出 `NullPointerException`；而 `set` 会安全进入 `else -> false`。
