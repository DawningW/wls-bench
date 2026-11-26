# 1-mmkv: Which function can be inlined and have its dead code eliminated?

## Answer: `set` (the version using `is` type checks)

Both functions are declared `inline` with `reified T`, so the Kotlin compiler inlines both at every call site. The difference is whether the dead branches can then be removed. Only `set` allows that.

### Why `set` works
- `when (value) { is Boolean -> ... }` tests the type of the value with `instanceof`. This is a plain type test, and it also fits `Parcelable`, because subclasses match.
- After inlining, the argument's concrete type is known at the call site. For example, `set("k", 1, 0)` passes a boxed `Integer`.
- `Integer instanceof Boolean`, `instanceof String` and the rest are therefore constant `false`. Only the `is Int` branch survives.
- The Kotlin compiler itself does not prune these branches. In practice the pruning is done by R8/D8 or the JIT, which fold the `instanceof` checks and drop the unreachable `encode` overloads and the `else -> false` branch.
- Each `encode` call is resolved statically to one overload, `encode(String, Int, Int)` and so on. With a known static type, the smart cast also removes the need for explicit casts.

### Why `set2` does not work well
- `T::class` is a reified `KClass` literal. Each `X::class ->` branch is compared with `==`. That becomes `Reflection.getOrCreateKotlinClass(...)` plus `Intrinsics.areEqual`, which are runtime calls and not compile-time constants.
- The optimizer does not fold them, so all branches stay in the bytecode and are evaluated at runtime.
- The `value as Boolean` and similar casts stay in the bytecode too.
- It is also semantically weaker. `Parcelable::class` is matched by exact equality, so a concrete `Parcelable` subclass (`T = MyParcel`) never matches and falls to `else -> false`. A nullable `T` and boxed/primitive class differences add further problems.

### Conclusion
`set` is the one that is inlined and then has its dead code removed. `set2` is inlined, but its `when (T::class)` branches are not removed.

Notes:
- `mmkv` must be public or `@PublishedApi internal`, because it is accessed from public inline functions.
- If true compile-time branch pruning is wanted, the option is to resolve the type once per call site, either with overloads or a type-class approach. With reified `T`, `value is X` is the idiomatic way.

---

> 以下为针对首轮两处模糊表述的无提示澄清追问（Follow-up）原始回答：

# Follow-up clarifications (static reasoning only; nothing was compiled or run)

## 1. `set2<Int>("k", 1, 0)` vs `set2<Int?>("k", null, 0)`

My earlier phrase "boxed/primitive class differences add further problems" was **imprecise and partly wrong**. The primitive/boxed difference is *not* a problem. The nullable case is a problem, but for a different reason.

**Comparison mechanism**
- `when (T::class) { Int::class -> ... }` is compiled as a `when` with a subject. Each branch tests `Intrinsics.areEqual(subject, Int::class)`, which calls `subject.equals(Int::class)`.
- Both sides are `KClass` objects made by `Reflection.getOrCreateKotlinClass(java.lang.Class)`.
  - `Int::class` is built from the primitive `Integer.TYPE` (`int`).
  - A reified `T = Int` may be built from `Integer.class`.
- Without kotlin-reflect, the object is a `ClassReference`. Its `equals` compares `javaObjectType` of both sides, so `int` and `Integer` are normalised to `Integer` and are equal.
- With kotlin-reflect, `KClassImpl.equals` does the same normalisation.
- So the primitive/boxed difference does not cause a mismatch.

**`set2<Int>("k", 1, 0)`**
- `T::class == Int::class` is **true** (match).
- The `Int::class ->` branch runs and calls `encode(key, value as Int, expire)`.
- This works normally.

**`set2<Int?>("k", null, 0)`**
- `T::class` for a reified `T = Int?` yields the class of `Int`. The nullability is erased in a class literal, because `KClass` has no nullable form.
- So `T::class == Int::class` is also **true** (match).
- The branch executes `value as Int` with `value == null`.
- At runtime this throws `NullPointerException` ("null cannot be cast to non-null type kotlin.Int"). It is a `NullPointerException` because the cast to non-null `Int` is a null check, not a `ClassCastException`.
- So the result is a crash, not a fall-through to `else`. `T::class` loses the nullability information, and `value as X` is unsafe when `T` is nullable.
- By contrast, `when (value) { is Int -> ... }` would not match `null` and would go to `else -> false`.

**Confidence**
- High on the equality semantics, because `ClassReference.equals` and `KClassImpl.equals` normalise through `javaObjectType`.
- High on the NPE.
- I have not run this. The exact bytecode for `Int::class` versus reified `Int` (`Integer.TYPE` or `Integer.class`) is not something I verified, but it does not change the outcome.

## 2. Does `kotlinc` prune the 8 branches of `set` when called as `set("k", 1, 0)` and `set("k", "abc", 0)`?

**Expected answer: all 8 branches stay in the `.class` bytecode, for both calls.**
- The `kotlinc` JVM backend inlines `inline` functions at the bytecode level.
- The argument is stored into a local variable of type `Object`: for example `Integer.valueOf(1)`, or the `"abc"` string.
- Each `is X` branch then compiles to a runtime `instanceof X` on that local, followed by `IFEQ`.
- Bytecode-level optimisations in `kotlinc` include:
  - redundant box/unbox elimination
  - null-check removal
  - removal of code made unreachable by constant branches (`ICONST` followed by `IFEQ`)
  - peephole clean-ups
- None of these folds `instanceof` against a value whose runtime class is only known from the preceding `Integer.valueOf` or constant string. So the `instanceof` chain, the `encode` overload calls and the `else` branch all remain.
- At runtime:
  - `set("k", 1, 0)` runs the `instanceof` checks in order until `is Int` and returns early.
  - `set("k", "abc", 0)` runs the checks until `is String`.
  - The remaining branches are present but not executed.

**What my earlier text should have said**
- "`Integer instanceof Boolean` is constant false" is a *semantic* fact that a later optimiser can exploit. It is not something `kotlinc` realises.
- Actual elimination depends on R8, or on C2/Graal JIT after profiling and type analysis. Even there, it is not guaranteed.

**Uncertainty**
- I did not compile or disassemble anything. This is based on my understanding of the JVM backend.
- If a Kotlin 2.x version uses the new IR-level inliner for JVM, or K2 folds `1 is Boolean` on a literal, the picture could differ slightly. I believe `kotlinc` still does not prune these branches, but I am not certain.
- To be sure, run `kotlinc` and then `javap -c` on the output.
