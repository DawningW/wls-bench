set（用 is 的那个）能被内联并做死代码消除；set2 只能内联，分支消不掉。

原因：

is 检查在调用点是编译期可判定的
inline + reified T 会在调用点把 T 代换成具体类型，value 的静态类型就确定了。set("k", "abc", 0) 内联后，value is Boolean、is Int …… 这些检查的类型关系编译器全知道，不可能命中的分支会被折叠掉，最后基本只剩一次 mmkv.encode(key, "abc", expire) 调用。

when (T::class) 是运行时对象比较
T::class 内联后生成的是一个运行时 KClass 实例（Reflection.factory().createKotlinClass(...) / ClassReference），Boolean::class == xxx 走的是 KClass.equals()。这不是编译期常量，编译器无法折叠，8 个分支连同 8 个 as 转换全部留在字节码里；而且未命中类型时要顺序跑完 8 次反射式比较才落到 else，比 set 慢得多。

set2 还有语义 bug
Parcelable::class 只匹配 Parcelable 这个类型本身，永远匹配不到具体实现类（MyData::class != Parcelable::class），所以那个分支是死代码，Parcelable 会静默走 else 丢数据。is Parcelable 才是对的。同理以后想支持 List/Set（MMKV 有对应重载）也只能用 is。

顺带：nullable 的行为差异
set("k", null, 0) 所有 is 都为 false → 静默返回，什么都没存（删值应该用 remove）。set2 则在 value as String 处直接抛 NPE。这也是 is 版本更安全的地方。

结论：保留 set，删掉 set2。
