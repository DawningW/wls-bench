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
actual inline fun <reified T> set2(key: String, value: T, expire: Int) = when (T::class) {
    Boolean::class -> mmkv.encode(key, value as Boolean, expire)
    Int::class -> mmkv.encode(key, value as Int, expire)
    Long::class -> mmkv.encode(key, value as Long, expire)
    Float::class -> mmkv.encode(key, value as Float, expire)
    Double::class -> mmkv.encode(key, value as Double, expire)
    String::class -> mmkv.encode(key, value as String, expire)
    ByteArray::class -> mmkv.encode(key, value as ByteArray, expire)
    Parcelable::class -> mmkv.encode(key, value as Parcelable, expire)
    else -> false
}.toUnit()
```

哪个函数能内联并进行死代码优化
