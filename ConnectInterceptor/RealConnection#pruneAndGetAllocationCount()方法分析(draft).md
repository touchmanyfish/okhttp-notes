# RealConnection#pruneAndGetAllocationCount()方法分析(draft)

```kotlin
private fun pruneAndGetAllocationCount(connection: RealConnection, now: Long): Int {
    connection.assertThreadHoldsLock()

    val references = connection.calls
    var i = 0
    while (i < references.size) {
      val reference = references[i]

      if (reference.get() != null) {
        i++
        continue
      }

      // We've discovered a leaked call. This is an application bug.
      val callReference = reference as CallReference
      val message = "A connection to ${connection.route().address.url} was leaked. " +
          "Did you forget to close a response body?"
      Platform.get().logCloseableLeak(message, callReference.callStackTrace)

       // 注释A 
      references.removeAt(i)
       // 注释B
      connection.noNewExchanges = true

      // 注释C
      // If this was the last allocation, the connection is eligible for immediate eviction.
      if (references.isEmpty()) {
        connection.idleAtNs = now - keepAliveDurationNs
        return 0
      }
    }

    // 注释D
    return references.size
  }

```
## 分析准备

```text
// 这里假设当前连接承载HTTP2.0
connection.calls:
  CallReference(null)
  CallReference(RealCall_1)
  CallReference(RealCall_2)
```

### calls中CallReference移除路径分析
```text
      .....
        │
        ▼
RealCall.callDone(...)
        │
        ▼
RealCall.releaseConnectionNoEvents()
        │
        ▼
connection.calls.removeAt(index)
```
如果要移除calls中的元素，最终会执行到上述移除路径。

所以，如果RealCall所在的CallReference还在calls中，说明对应的RealCall#callDone()方法还未被调用。

### callDone调用条件分析

```text
RealCall:
 requestBodyOpen
 responseBodyOpen
 expectMoreExchanges
 
// callDone调用条件 
callDone = !requestBodyOpen && !responseBodyOpen && !expectMoreExchanges
```

结合上一小节有如下结论：
```text
RealCall所在CallReference没被移除 
    -> callDone()没被调用 
    -> callDone()调用条件至少有一个没被满足
```

## 分析尝试1(没有分析成功：TODO)

### 注释A分析

TODO

### 注释B分析

分析思路：
```text
举特殊例子
->导致CallReference(null)
->此时connection不可用
->如果换其他例子此时connection可能不可用
->okhttp没有足够证据证明connection没问题
->于是保守的认为connection有问题
->于是立即将connection标记为不可用+当前还未完成的流继续跑
```

我们只要证明当CallReference(null)时可能出问题，就能理解OkHttp的对应处理。

**HTTP1.1下有问题的情况:**

```text
// 出错举例
POST+请求发送一半报错
```
请求发送一半时responseBodyOpen必然为true，于是callDone()没有被调用，对应CallReference(RealCall)
不会被移除；如果RealCall没有被任何非弱引用持有，它会被垃圾回收。这证明了这个例子会导致CallReference(null)。

另外显然，一个1.1连接上如果请写一半报错，这个连接是没法继续使用的。这证明了此时connection不可用。

**HTTP2.0下有问题的情况:**

TODO:我目前找不到HTTP2.0下的特殊例子。

**于是这种分析思路就卡在这个地方。**

### 注释C分析

TODO

### 注释D分析

TODO

## 分析尝试2(TODO:这是一个不精确的分析)

### 注释A分析

CallReference(null)意味着它原本持有的RealCall被垃圾回收了，留着它没有意义，所以将其移除了。

### 注释B猜测(TODO)

下面是我猜的.

CallReference(null)意味着被回收call上的相关信息丢失了，对应connection的可用性于是就处于一个未知的状态。于是OkHttp选择放弃这个connection.

### 注释C分析

connection在B处已经被标记为不可复用了，C处又检测到关联的call没有了，这个connection还留着干什么？于是手动将这个“已不可被复用的connection”
设置为满足被清理的条件，方便cleanup后续对它进行清理。

### 注释D猜测(TODO)

对于CallReference(RealCall)有：

1. callDone()没有被调用
2. RealCall可能已经不被非弱引用持有了，只是还没来得及被回收
3. RealCall被非弱引用持有

情况3包含：
* call在执行过程中正常被非弱引用持有 
* call已经执行完毕了还被非弱引用错误持有

OkHttp将情况2+情况3的数目作为“connection上存活call的数目”，并且在cleanup()如果发现这个数目 > 0则认为connection正在被使用。这显然是一个
不精确的结论。




