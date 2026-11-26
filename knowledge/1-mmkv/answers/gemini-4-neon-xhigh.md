# 结论

- **两个函数都能被 Kotlin 编译器（`kotlinc`）内联**（均声明为 `inline`）。
- **只有 `set`（使用 `when (value) { is ... }`）能够进行有效的死代码优化（DCE）**；**`set2`（使用 `when (T::class)`）无法进行死代码优化**。
- 需要明确区分编译阶段：**`kotlinc` 本身在生成 JVM 字节码时，对 `set` 和 `set2` 都不会消除 `when` 中的死分支**；死代码优化发生在后续的 **R8（Android Release 优化/裁剪阶段）** 或运行时 **ART / HotSpot C2 JIT** 阶段。

---

# 原理分析

题目中出现 `Parcelable` 和 `mmkv`，属于典型的 Kotlin Multiplatform（Android/JVM 目标）场景，完整编译链路为：`kotlinc`（JVM 字节码） → D8/R8（DEX 优化） → ART/JVM 运行时。

## 1. 为什么 `set` 能进行死代码优化

```kotlin
actual inline fun <reified T> set(key: String, value: T, expire: Int) = when (value) {
    is Boolean -> mmkv.encode(key, value, expire)
    is Int -> mmkv.encode(key, value, expire)
    is Long -> mmkv.encode(key, value, expire)
    is Float -> mmkv.encode(key, value, expire)
    is Double -> mmkv.encode(key, value, expire)
    is String -> mmkv.encode(key, value, expire)
    is ByteArray -> mmkv.encode(key, value, expire)
    is Parcelable -> mmkv.encode(key, value, expire)
    else -> false
}.toUnit()
```

1. **`kotlinc` 阶段**：
   - `set` 会被内联到调用点，展开为针对 `value` 的一连串 `instanceof` 与 `checkcast` 指令。
   - 注意：`kotlinc` 自身不会在这里做 `instanceof` 常量折叠，因此查看 `.class` 字节码时，所有 8 个分支仍然存在。
   - 另外，`set` 函数体并没有真正使用 `reified T`（没有 `T::class`、`is T` 等操作）；让调用点具体类型得以暴露给下游优化器的是 **`inline`** 而非 `reified`。

2. **R8 / ART / JIT 阶段（真正完成 DCE）**：
   - 由于已经内联到调用点，`value` 的静态类型在调用处通常是确定的：
     - **引用类型（如 `String`、`ByteArray`）**：例如 `set("k", "abc", 0)` 或非空 `name: String`，R8 知道 `value` 具有精确非空类型 `java.lang.String`（`final` 类）。
       - `instanceof Boolean / Integer / Long / Float / Double` 等互斥类型直接静态折叠为 `false`，分支被移除；
       - `instanceof String` 静态折叠为 `true`，类型检查本身被消除，其后的 `ByteArray`、`Parcelable`、`else` 分支均变为不可达代码而被彻底消除；
       - 多余的 `(String)` 强转也被移除。最终等价于直接调用 `mmkv.encode("k", value, 0)`。
     - **`Parcelable` 实现类（如 `class User : Parcelable`）**：`Boolean` ~ `byte[]` 均是 `final` 类或数组，与 `User` 互斥，前 7 个 `instanceof` 全部折叠为 `false` 并消除；`user instanceof Parcelable` 在非空时折叠为 `true`，`else` 被消除，同样简化为直接调用 `mmkv.encode(key, user, expire)`。
     - **基本类型（如 `Int`、`Boolean` 等）**：因泛型参数 `value: T` 在 JVM 字节码中擦除为 `Object`，`kotlinc` 内联前会插入装箱调用（如 `Integer.valueOf(x)`）。R8 明确知道 `Integer.valueOf` 返回非空且精确类型为 `java.lang.Integer`，因此同样能将其他 7 个分支和 `else` 分支全部消除，并移除多余的 `CHECKCAST Number`。不过装箱/拆箱本身（`Integer.valueOf(x)` → `intValue()`）可能因 R8 版本不同而保留在 DEX 中，或在后续由 R8/ART JIT 逃逸分析进一步消除。
     - **可空具体类型（如 `String?`）**：不匹配的 7 个分支仍会被全部消除，只保留等价于 `if (value != null) mmkv.encode(key, value, expire)` 的空检查。
     - **不支持的 `final` 具体类型**：8 个 `instanceof` 全部折叠为 `false`，所有 `encode` 分支全部被消除。
   - **唯一无法完全消除分支的情况**：调用点传入的静态类型就是宽泛父类型（如 `Any`/`Any?`），且 R8 的跨过程类型传播也无法将其收窄为具体类型（此时运行时分发本来就是必需的）。

---

## 2. 为什么 `set2` 无法进行死代码优化

```kotlin
actual inline fun <reified T> set2(key: String, value: T, expire: Int) = when (T::class) {
    Boolean::class -> mmkv.encode(key, value as Boolean, expire)
    Int::class -> mmkv.encode(key, value as Int, expire)
    ...
    Parcelable::class -> mmkv.encode(key, value as Parcelable, expire)
    else -> false
}.toUnit()
```

1. **`kotlinc` 阶段**：
   - `set2` 同样会被内联，内联时 `T::class` 中的 `T` 会被替换为调用点的具体类型。
   - 但是，`when (T::class)` 并不是编译期常量匹配，在 JVM 字节码中会展开为类似如下逻辑：
     ```java
     KClass var1 = Reflection.getOrCreateKotlinClass(String.class);
     if (Intrinsics.areEqual(var1, Reflection.getOrCreateKotlinClass(Boolean.TYPE))) {
         mmkv.encode(key, (Boolean) value, expire);
     } else if (Intrinsics.areEqual(var1, Reflection.getOrCreateKotlinClass(Integer.TYPE))) {
         mmkv.encode(key, ((Number) value).intValue(), expire);
     } ...
     ```
   - `kotlinc` 不会对 `KClass` 相等比较做常量折叠，所有分支全部保留。

2. **R8 / JIT 阶段依然无法消除死代码**：
   - **`Reflection.getOrCreateKotlinClass` 是动态工厂调用**：其内部委托给静态字段 `Reflection.factory`，该字段在类初始化时通过 `Class.forName("kotlin.reflect.jvm.internal.ReflectionFactoryImpl")` 动态加载；未引入 `kotlin-reflect` 时回退到 `ReflectionFactory`，每次调用都会 `new ClassReference(javaClass)` 创建新对象（引入时则查反射缓存）。
   - **`Intrinsics.areEqual` 触发复杂的虚调用**：底层调用 `ClassReference.equals(other)`，内部还会调用 `javaObjectType`（通过 `Class.isPrimitive()`、`Class.getName()` 等将基本类型 `Class` 映射为装箱 `Class`，以确保 `int.class` 与 `Integer.class` 对应的 `KClass` 相等）。
   - 由于涉及反射动态工厂、对象分配、接口虚分发以及 JDK 运行时方法调用，**R8 无法静态证明 `Intrinsics.areEqual(Reflection.getOrCreateKotlinClass(A), Reflection.getOrCreateKotlinClass(B))` 的结果恒为 `true` 或 `false`**。
   - 结果：
     1. **所有 8 个分支和 `else` 都会完整残留在最终 DEX 中**，每个调用点都被内联膨胀出整套分支链；
     2. 每次调用都在运行时重复执行 `Reflection.getOrCreateKotlinClass` 和 `Intrinsics.areEqual`，带来额外的对象分配与性能开销；
     3. R8 也无法对项目中未用到的 `mmkv.encode(...)` 重载进行裁剪。

---

# 对比总结

| 维度 | `set` (`when (value) { is ... }`) | `set2` (`when (T::class)`) |
| --- | --- | --- |
| `kotlinc` 能否内联 | 能 | 能 |
| `kotlinc` 阶段死代码消除 | 否（仍保留全部 `instanceof` 分支） | 否（仍保留全部 `KClass` 比较分支） |
| R8 / JIT 阶段死代码消除 | **能**（具体类型下可折叠 `instanceof` 并移除全部死分支） | **否**（`Reflection.getOrCreateKotlinClass` + `Intrinsics.areEqual` 无法静态求值） |
| 内联后 Release DEX 体积 | 极小（折叠为单个 `mmkv.encode` 调用） | 严重膨胀（每个调用点保留全部 8 个分支） |
| `Parcelable` 子类支持 | 正确（`is` 检查继承/实现关系） | **有 Bug**（见下文） |

---

# 额外需要警惕的 `set2` 语义问题

除了无法进行死代码优化外，`set2` 还存在明显的正确性隐患：

1. **`Parcelable` 子类会静默写入失败**：
   `when (T::class)` 做的是**精确类相等**比较，而不是子类型判断。传入具体实现类 `User : Parcelable` 时，`T` 推断为 `User`，`User::class != Parcelable::class`，会直接落入 `else -> false`（除非调用方手动写 `set2<Parcelable>(...)`）。而 `set` 的 `is Parcelable` 基于 `instanceof`，可以正确匹配任意子类。
2. **变量静态类型为 `Any` 时失效**：
   例如 `val v: Any = "hello"; set2("k", v, 0)`，`T::class` 为 `Any::class`，无法命中 `String::class`，会直接走 `else -> false`；而 `set` 仍可在运行时正确命中 `is String`。
3. **可空类型的 `null` 值会崩溃**：
   若传入 `val s: String? = null`，`set` 中 `null is String` 为 `false`，安全返回 `false`；而 `set2` 中 `T::class` 仍为 `String::class`，执行 `value as String` 会在运行时抛出 `NullPointerException`。

---

# 工程建议

- 若希望在 **编译期（包括 Debug 构建和无 R8 环境）** 就完全消除多余分支、避免基本类型装箱，并在编译阶段拦截不支持的类型，最优方案是直接为支持的 8 种类型编写**重载函数**，而非使用泛型 `inline` 函数分发。
- 如果受限于统一的泛型 API 形态必须在两者之间二选一，应选择 **`set`**（配合 R8 开启优化即可完成死代码消除）。

---

> 以下为第二轮独立复测（Round 2）原始回答：

## 结论

- **内联（Inline）**：**`set` 和 `set2` 都能被 Kotlin 编译器（`kotlinc`）内联**（两者均为 `actual inline fun <reified T>`，Kotlin 调用处都会展开函数体）。
- **死代码优化（DCE / 分支消除）**：
  - **仅靠 `kotlinc`（编译到 `.class` JVM 字节码阶段）**：**两者都不会做分支死代码消除**，`kotlinc` 只负责内联展开和 `reified T` 替换，不会对 `instanceof` 或 `KClass` 比较进行常量折叠，`.class` 中均会保留全部 8 个分支。
  - **经过 R8（Android Release 构建的字节码优化器，以及运行时 ART/JVM JIT）**：**只有 `set` 能进行死代码优化（消除不可达分支），`set2` 不能。**

因此，在题目对应的 Android（MMKV + `Parcelable`）实际构建场景下，答案是 **`set`**。

---

## 原理分析

### 1. 为什么 `set` 能在 R8（及 JIT）下完成死代码消除

以调用 `set("k", "hello", 0)` 和 `set("k", 123, 0)` 为例。

`kotlinc` 内联 `set` 后，调用处生成的等价字节码为：

```java
// set("k", "hello", 0)
Object value$iv = "hello"; // 实际产生的 SSA 值类型为非空 java.lang.String
if (value$iv instanceof Boolean) {
    mmkv.encode("k", ((Boolean) value$iv).booleanValue(), 0);
} else if (value$iv instanceof Integer) {
    mmkv.encode("k", ((Integer) value$iv).intValue(), 0);
} ...
else if (value$iv instanceof String) {
    mmkv.encode("k", (String) value$iv, 0);
} else if (value$iv instanceof byte[]) {
    mmkv.encode("k", (byte[]) value$iv, 0);
} else if (value$iv instanceof Parcelable) {
    mmkv.encode("k", (Parcelable) value$iv, 0);
} else {
    // false
}
```

R8 对调用处做类型传播与 `instance-of` 优化时：

1. **静态精确类型已知**：内联把调用处的实参直接暴露给了 `when (value)` 的 `instanceof` 链。
   - 对于 `"hello"`（或非空 `String` 参数），R8 知道 `value$iv` 是非空 `java.lang.String`。
   - 对于 `123`（基本类型 `Int`），`kotlinc` 会先装箱成 `Integer.valueOf(123)`，R8 知道 `Integer.valueOf(...)` 返回的是非空 `java.lang.Integer`。
   - 对于 `ByteArray` 或具体实现类 `MyParcelable`，R8 同样知道其具体类型。
2. **`instanceof` 常量折叠**：`Boolean`、`Integer`、`Long`、`Float`、`Double`、`String`、`byte[]` 在 JVM/DEX 中全都是 **`final` 类型**（不能被继承，也不实现 `Parcelable`）。
   - 所有不匹配的 `instanceof` 检查都会被 R8 直接折叠为常量 `false`，对应分支成为不可达代码并被删除。
   - 命中的 `instanceof` 检查（在确认非空时）被折叠为常量 `true`，其后所有剩余分支（含 `else`）全部不可达并被删除；同时分支内的 `check-cast` 冗余强转也被消除。
   - 即使实参是可空类型（如 `String?`），其余不兼容分支仍会被全部消除，仅保留 `if (v != null) mmkv.encode(...)`。
3. **最终效果**：
   - `set("k", "hello", 0)` 会被 R8 精简为单条 `mmkv.encode("k", "hello", 0)`。
   - `set("k", 123, 0)` 会消除全部 7 个死分支和所有 `instanceof`/`check-cast`，仅保留匹配分支的 `mmkv.encode(...)`（注：由于泛型参数 `value: T` 擦除为 `Object`，`kotlinc` 会在内联前插入 `Integer.valueOf`、分支内调用 `.intValue()`，是否进一步消除这组单次装箱/拆箱取决于 R8 版本及 ART JIT 逃逸分析）。
   - **前提边界**：如果调用处传入的变量静态类型本身就是宽泛类型（如 `val x: Any = ...; set("k", x, 0)`），编译期无法确定实际类型，此时 R8 也无法消除分支。

---

### 2. 为什么 `set2` 不能进行死代码消除

`set2` 使用的是 `when (T::class)`，比较的是 Kotlin 的 **`KClass`**（`kotlin.reflect.KClass`），而不是可直接静态判定的 `instanceof`。

以 `set2("k", 123, 0)` 为例，`kotlinc` 内联并替换 `reified T` 后生成的等价字节码为：

```java
KClass tmp = Reflection.getOrCreateKotlinClass(Integer.class); // reified T 擦除为包装类 Integer.class
if (Intrinsics.areEqual(tmp, Reflection.getOrCreateKotlinClass(Boolean.TYPE))) {
    mmkv.encode("k", ((Boolean) value$iv).booleanValue(), 0);
} else if (Intrinsics.areEqual(tmp, Reflection.getOrCreateKotlinClass(Integer.TYPE))) {
    mmkv.encode("k", ((Integer) value$iv).intValue(), 0);
} ...
```

R8（及 `kotlinc`）无法对 `set2` 做死代码消除的根本原因在于：

1. **经过动态反射工厂 `Reflection.getOrCreateKotlinClass`**：
   - `kotlin.jvm.internal.Reflection` 的静态字段 `factory` 在 `<clinit>` 中通过 `Class.forName("kotlin.reflect.jvm.internal.ReflectionFactoryImpl")` 动态初始化（未引入 `kotlin-reflect` 时回退到 `ReflectionFactory`，每次调用 `new ClassReference(cls)`；引入 `kotlin-reflect` 时通过缓存表返回 `KClassImpl`）。
   - R8 无法在编译期静态求值这类动态反射工厂及对象构造/缓存查找。
2. **`when` 使用 `Intrinsics.areEqual` 调用 `ClassReference.equals`**：
   - `ClassReference.equals` 并非简单的引用比较，而是比较 `javaObjectType == other.javaObjectType`。
   - 注意一个关键细节：对于基本类型（如 `Int`），`reified T` 对应的 `T::class` 传入的是包装类 `Integer.class`，而分支中的 `Int::class` 传入的是原始类型 `Integer.TYPE`（`int.class`），两者底层 `jClass` 甚至不是同一个 `Class` 对象！
   - `javaObjectType` 内部需要调用 `jClass.isPrimitive()`（JDK/Android 框架方法）并查询 `ClassReference` 静态初始化构建的 `PRIMITIVE_CLASSES` `Map` 来做原始类型到包装类的归一化映射。
   - R8 无法静态折叠 `Class.isPrimitive()` 与 `HashMap.get()`，因此 **无法判定任何一个 `Intrinsics.areEqual` 的真假**。
3. **最终后果**：
   - `set2` 内联后不仅**无法消除任何死分支**，反而会把全部 8 个分支连同大量 `Reflection.getOrCreateKotlinClass`、`Intrinsics.areEqual` 调用一起复制到每一个调用点，造成明显的**代码膨胀（code bloat）和运行时额外开销**。

---

## 补充：`set2` 存在的额外语义缺陷与更优实践

1. **`set2` 的功能性 Bug**：
   - **`Parcelable` 子类全部失效**：`Parcelable` 是接口，调用 `set2("k", myParcelable, 0)` 时 `T` 推导为具体子类 `MyParcelable`，`MyParcelable::class == Parcelable::class` 为 `false`，会直接落入 `else -> false` 导致写入失败；而 `set` 的 `is Parcelable` 能正常匹配子类，且 R8 仍可消除前面 7 个 `final` 类型分支。
   - **可空类型会抛 NPE**：调用 `set2("k", nullableInt, 0)` 且值为 `null` 时，`T::class` 仍为 `Int::class`，进入分支后执行 `value as Int` 会抛出 `NullPointerException`；而 `set` 中 `null is Int` 为 `false`，会安全落入 `else -> false`。
   - **父类引用失效**：`val v: Any = "abc"; set2("k", v, 0)` 中 `T` 为 `Any`，会落入 `else -> false`；而 `set` 可按运行时类型正确写入。
2. **即便改成 `when (T::class.java)` 也有陷阱**：
   - 同样无法解决 `Parcelable` 子类和 `null` 的问题；且对于 `Int` 等基本类型，`T::class.java` 得到的是 `Integer.class`，而 `Int::class.java` 得到的是 `int.class`（`Integer.TYPE`），两者不相等，会导致基本类型分支直接失效（除非全改为 `::class.javaObjectType`）。
3. **工程上更推荐的写法**：
   - 如果希望在**编译期就彻底零开销、无装箱、无内联膨胀，并获得编译期类型检查**（非法类型直接编译报错而非静默返回 `false`），直接为支持的 8 种类型提供显式**函数重载**（`set(key, value: Int, expire)` 等）优于这两种 `inline <reified T>` + `when` 写法。
