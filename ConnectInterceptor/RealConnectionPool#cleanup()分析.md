# RealConnectionPool#cleanup()分析.md

## cleanup()作用

cleanup()每次只会 **清理一个** 被粗略判断为idle并切符合evict条件的connection；cleanup()会返回下一次出现符合evict条件的connection的
最快时间x。cleanup()会被安排在x后再次执行。

## 粗略判断是否idle

pruneAndGetAllocationCount()方法返回当前connection上还在执行的call数目。不过这是一种粗略的判断，这导致cleanup()
得出的是否idle结果也是
粗略的。

```kotlin
// RealConnectionPool#cleanup()

// 粗略判断
pruneAndGetAllocationCount(connection, now) > 0
```

## 执行流程

分为四步：

1. 找出longestIdleConnection
2. 如果这个conncetion符合evict条件，清理它，然后立即进行下一轮cleanup()
3. 当前没有符合evict条件的connection，计算一下最快可能出现符合evict条件的connection的时间x，安排cleanup()在x后执行。
4. 当前没有任何connection，当前cleanup()不在导致任何后续cleanup()

### evict条件

1. longestIdleDurationNs >= keepAliveDurationNs
2. idleConnectionCount > maxIdleConnections

evict = 1 || 2

### evict connection

在cleanup()
单次执行中，OkHttp只判断了“当前是否至少有一个符合evict条件的connection”，OkHttp选择每次只evict一个，然后安排立执行(**return
0**)cleanup()
处理下一个。

### 计算最快可能出现符合evict条件的connection的时间x

如果当前有idle的connection，那么它们最快可能出现符合evict条件的情况为：

```text
此时idle时间最长的一直保持idle直到：
keepAliveDurationNs - longestIdleDurationNs
```

如果当前全是inUse的connection，那么它们最快可能出现符合evict条件的情况为：

```text
假设其中一个connection立即进入idle状态并保持至少keepAliveDurationNs
```

如果当前没有任何conntion了，使用-1表示"本次cleanup()执行不再导致新的cleanup()执行"

注意，当时间经过了x也不代表一定有符合evict条件的connection出现！OkHttp选择安排一次x(**return x**)时间后的cleanup()执行：

1. 如果此时有满足evict条件的，开心执行evict
2. 如果没有则继续计算x并安排cleanup(),此时会有些许性能损耗(局部加锁)。

这里可以看出OkHttp用些许性能损耗换了对evict条件的尽快检测

## 加锁分析

```kotlin
synchronized(connection) {
    // If the connection is in use, keep searching.
    if (pruneAndGetAllocationCount(connection, now) > 0) {
        inUseConnectionCount++
    } else {
        idleConnectionCount++

        // If the connection is ready to be evicted, we're done.
        val idleDurationNs = now - connection.idleAtNs
        if (idleDurationNs > longestIdleDurationNs) {
            longestIdleDurationNs = idleDurationNs
            longestIdleConnection = connection
        } else {
            Unit
        }
    }
}

synchronized(connection) {
    if (connection.calls.isNotEmpty()) return 0L // No longer idle.
    if (connection.idleAtNs + longestIdleDurationNs != now) return 0L // No longer oldest.
    connection.noNewExchanges = true
    connections.remove(longestIdleConnection)
}
```

这里的加锁模式是：
1. 缩小锁的范围，只对状态加锁；
2. 根据状态做出计算结果，但同时不修改其他状态。接着释放锁
3. 一些其他计算。
4. 再次对状态加锁，判断被锁状态是否变化，如果没变则使用2步骤中的计算结果。如果变化则从头开始执行。

这种模式的好处是通过降低锁力度+重试来提高并发性能，但同时可能会伴随计算量的增加。
