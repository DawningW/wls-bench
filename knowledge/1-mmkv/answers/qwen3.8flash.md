没有本地 kotlinc，所以以下是基于 Kotlin 编译规则的分析（要实测得装 kotlinc 后反编译调用点字节码）。

结论

只有 set2 的分支选择是编译期决定的，set 一定是运行时分派。但两者都别指望 Kotlin 前端真的把代码折成一条分支。

set — when (value)
判别对象是参数的运行时类型。即使 T 是 reified、调用点静态类型已知（set(...)），字节码里仍是一串 instanceof，8 个分支全部保留。而且它压根不看 T：set(k, 1, 0) 走的是 Int 分支——好处是按实际对象分派，坏处是无法消除死代码。

set2 — when (T::class)
判别对象是类型参数，inline 后 T::class 被替换成调用点的类型常量，条件对每个调用点是常量、与运行时值无关，所以 JIT / R8 这类做常量传播的优化器可以把未选中分支折掉。两个限制：

Kotlin/JVM 后端把 T::class == X::class 编成 Intrinsics.areEqual(Reflection.getOrCreateKotlinClass(...))，字节码层面分支还在，折叠发生在 JIT 层。别用 checkcast 数量去推断"已经只剩一个调用"。
精确类型匹配带来的 bug：

set(key, myThing, 0)   // MyThing::class != Parcelable::class → else → false

Parcelable 分支永远只在 T 恰好是 Parcelable 接口本身时命中，业务代码几乎不会这样写。set 的 value is Parcelable 反而能匹配子类，是对的。

建议

要真正得到"只有一条分支、零死代码"，不要用泛型 inline 做分派，改用重载——解析期就选定了函数，字节码里不存在其他分支，MMKV 自身的 API 就是这个形状：

actual fun set(key: String, value: Int, expire: Int) { mmkv.encode(key, value, expire) }
actual fun set(key: String, value: String, expire: Int) { mmkv.encode(key, value, expire) }
// ...
expect fun set(key: String, value: Parcelable, expire: Int)  // 只在 android actual 里提供

如果必须保留单个泛型入口（common 侧签名限制），就把两者结合——用 T 走快路径、用运行时类型兜底：

actual inline fun  set(key: String, value: T, expire: Int) = when {
    value is Boolean -> mmkv.encode(key, value, expire)
    // ... 基础类型分支
    value is Parcelable -> mmkv.encode(key, value, expire)
    else -> false
}.toUnit()

这样 is 检查同时拿到"按实际类型分派"的正确性，成本是每次调用一组 instanceof（分支顺序按使用频率排，命中的放前面）。

需要我装个 kotlinc 把两种写法在调用点的字节码反编译出来对比吗？能直接看清 when (T::class) 到底有没有被折掉。
