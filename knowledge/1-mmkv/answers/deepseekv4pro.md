两个函数都能 `inline`，但**只有 `set2` 能真正进行死代码优化（dead code elimination）**。

关键在于 `when` 判断的对象不同。

## 核心区别

| | `set` | `set2` |
|---|---|---|
| `when` 判断什么 | 运行时值 `value` | 编译期类型 `T::class` |
| 分支能否被编译器裁剪 | ❌ 不能 | ✅ 能 |

---

### `set`：判断的是运行时对象

```kotlin
when (value) {   // value 是运行时值，类型在编译期无法确定
    is Boolean -> ...
    is Int -> ...
    ...
    else -> false
}
```

`is Boolean`、`is Int` 这些是**类型检查**，针对的是运行时的具体值。即使 `T` 被 reified，编译器也只知道 `T` 的具体类型，但 `value` 变量本身的类型仍是 `T`，编译器**无法在编译期消掉其他分支**——因为 `is` 检查在运行时才发生。所以每个分支的代码都会保留。

---

### `set2`：判断的是编译期已知的类型

```kotlin
when (T::class) {   // T 被 reified，T::class 是编译期常量
    Boolean::class -> mmkv.encode(key, value as Boolean, expire)
    Int::class -> ...
    ...
}
```

`T::class` 在编译期就是确定的值。当调用 `set2("k", true, 1)` 时，编译器会把 `T` 替换成 `Boolean`，于是：

- `T::class` → `Boolean::class`
- `when` 变成了 `Boolean::class` 与各分支的 `::class` 比较，**编译期可判定**，只保留 `Boolean::class` 这一支
- 其他分支（Int、Long、String…）成为死代码被消除
- `value as Boolean` 也变成了安全的直接转换

**这就是典型 reified + `when(T::class)` 的写法目的**——让编译器为每种类型生成特化版本，并裁剪掉无用分支。

---

## 结论

- **`set2` 能内联并做死代码优化**：`T::class` 是编译期常量，`when` 分支可静态求值，无用的分支会被消除。
- **`set` 只能内联，不能做死代码优化**：`when(value)` 是运行时类型检查，编译器保留全部分支。

补充一点：虽然 `set2` 在死代码消除上更优，但实际很多 MMKV/KV 封装里 `set` 这种写法更常见，因为 `is` 智能转换语义清晰、可读性好，性能差异在绝大多数场景下可忽略。只有在追求极致内联展开、且类型集合固定的场景下，`set2` 的写法才有实际收益。
