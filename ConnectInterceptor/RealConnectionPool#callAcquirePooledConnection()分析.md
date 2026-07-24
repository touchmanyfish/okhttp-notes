# RealConnectionPool#callAcquirePooledConnection()分析

```kotlin
fun callAcquirePooledConnection(
    address: Address,
    call: RealCall,
    routes: List<Route>?,
    requireMultiplexed: Boolean
): Boolean {
    for (connection in connections) {
        synchronized(connection) {
            if (requireMultiplexed && !connection.isMultiplexed) return@synchronized
            if (!connection.isEligible(address, routes)) return@synchronized
            call.acquireConnectionNoEvents(connection)
            return true
        }
    }
    return false
}
```

## 作用

用于从pool中为call捞取已存在的connection。

## 参数分析

OkHttp在为call在池中捞取connection时会通过如下2个参数控制查找范围。

### routes

**非空值:** 在查找时开启http2.0请求合并验证；这会扩大搜索范围。

**null:** 表示本次查找不进行http2.0请求合并验证；这会减小搜索范围。

### requireMultiplexed

**true:** 表示只在http2.0 connection中进行查找。这会减小搜索范围。

**false:** 表示在所有版本的connection中进行查找。这会扩大搜索范围。
