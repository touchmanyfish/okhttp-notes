# RetryAndFollowUpInterceptor


## 拿到response以后的重试

拿到response，判断code，需要重试则生成request，发起重试，没啥好说的这里。


## 请求-响应过程中报错在理论上的可重试判断

### 粗略判断方法分析

```text
// 幂等（Idempotent）：
GET、HEAD、OPTIONS、TRACE、PUT、DELETE

// 非幂等（Non-Idempotent）：
POST、PATCH

// 流程
[OkHttp接收请求][建立连接][client发送请求][server处理请求][client读物响应]
```

**目标：**

重试以后不会在服务器产生副作用

**对于幂等方法：**

只要报错就放心大胆的去重试，直接自然的完成目标。

**对于非幂等方法：**

1. 如果异常在[client发送请求]或者之后的阶段抛出，服务端可以已经开始处理收到的请求数据了(可能会导致副作用产生)；于是如果要发起重试，异常至少要在
这个阶段之前抛出。
2. 另外，虽然[client发送请求]之前阶段抛出异常可以100%确定服务器还未产生副作用，但是有些异常表示一定不能重试(例如证书异常或者pinner异常),需要判断一下。

### 伪代码

于是现在我们有一个简陋的判断是否重试的方法：

```kotlin
// 判断当前阶段是否位于[client发送请求]或者之后
val requestSendStarted = ...
// 请求-响应阶段抛出的异常
val e =...

if (method.isIdempotent()) {
    return true
}

if (!method.isIdempotent() && !requestSendStarted) {
    return e.canRecover()
}

fun IOException.canRecover():Boolean{
    // 实现方式：
    // 1. 把整个过程中出现的可以发起重试的异常都列举出来
    // 2. return this in 可重试异常列表
}
```

### 实现过程中的问题

**问题1:不可能实现canRecover()**

你没法“把整个过程中出现的可以发起重试的异常都列举出来”，原因如下：

1. OkHttp无法控制运行时，所以不可能提前预测完整的运行时异常
2. OkHttp后续版本或者底层依赖的组件后续可能会新增异常类型，这个方法会错判。

**问题2:requestSendStarted难以精确判断**

可以通过设置一个状态来表示“请求是否已经开始了发送”，但是这要求你在所有“开始发送”的地方维护这个状态。

或者尝试使用“先将发送之前能抛出的所有异常列举出来，然后判断异常是否在其中”？根据前面的分析，这是不可能的，因为这个异常列表你没法穷举出来。

## 请求-响应过程中报的重试

### 实现特点

根据前面的分析，想要做到“精确判断请求是否开始发送”不是一个简单的问题。于是当请求-响应过程中抛出异常时，OkHttp选择：

1. OkHttp只处理部分明确不可恢复的场景，其余情况在满足条件时尝试恢复
2. 不关心请求是否在rfc中定义为非幂等，只关心这个请求的body是否是可以重复创建(isOneShot() == true)
3. 只在有明确证据时才确定当前的位于的阶段，否则就消极的/防御的认为当前处于“请求数据已经开始发送的阶段”
4. 显然1+2+3无法保证method在语义上的正确性，正确性此时由OkHttp的调用者来保证。

### 分析特点3

```kotlin
 try {
    response = realChain.proceed(request)
    newExchangeFinder = true
} catch (e: RouteException) {
    // 明确证据RouteException
    
    // The attempt to connect via a route failed. The request will not have been sent.
    if (!recover(e.lastConnectException, call, request, requestSendStarted = false)) {
        throw e.firstConnectException.withSuppressed(recoveredFailures)
    } else {
        recoveredFailures += e.firstConnectException
    }
    newExchangeFinder = false
    continue
} catch (e: IOException) {
    // An attempt to communicate with a server failed. The request may have been sent.
    
    // 明确证据:ConnectionShutdownException
    if (!recover(e, call, request, requestSendStarted = e !is ConnectionShutdownException)) {
        throw e.withSuppressed(recoveredFailures)
    } else {
        recoveredFailures += e
    }
    newExchangeFinder = false
    continue
}
```
#### 明确证据:RouteException

RouteException是一个连接过程中抛出的异常，所以当拦截到这个异常时，请求到数据显然还未发送。

#### 明确证据:ConnectionShutdownException

```kotlin
// 调用处1: Http2Connection#newStream()
if (nextStreamId > Int.MAX_VALUE / 2) {
    shutdown(REFUSED_STREAM)
}

if (isShutdown) {
    throw ConnectionShutdownException()
}

// 如果抛出异常 -> 不会分配流id -> 不会执行请求数据发送
streamId = nextStreamId
nextStreamId += 2
stream = Http2Stream(streamId, this, outFinished, inFinished, null)

// 调用处2:Http2Connection.kt
// 这个方法在OkHttp中没有被使用！
fun setSettings(settings: Settings) {
    synchronized(writer) {
        synchronized(this) {
            if (isShutdown) {
                throw ConnectionShutdownException()
            } okHttpSettings . merge (settings)
        } writer . settings (settings)
    }
}
```
在OkHttp中，ConnectionShutdownException只在上述两处抛出，而第二出还没有被使用，于是有如下结论：

* 当前触发该 ConnectionShutdownException 的请求数据确定没有开始发送。

让我们回到RetryAndFollowUpInterceptor中拦截到IOException的地方，此时有：

1. e is ConnectionShutdownException: OkHttp确定请求数据没被发送
2. e !is ConnectionShutdownException: OkHttp此时不能确定请求数据是否被发送，于是选择消极的/防御性的认为请求发送了

### 分析特点4

```text
// 漏判的例子
// 特定请求

isOneShot() == false
POST
请求体发送第3个字节时人为报错
```

对于这种请求，OkHttp会判定为可重试(isRecoverable()返回true)，不过这个请求如果需要的幂等性，则需要OkHttp的调用者自己来保证。

### 具体重试流程

如果在请求-响应阶段报错，是否能重试判断流程如下：

1. 用户开启了重试(client.retryOnConnectionFailure)吗？
2. 请求body是否还能再次发送？
3. 通过异常+requestSendStarted判断是否可以恢复
4. 1/2/3通过，此时会进入route选择流程(TODO:旧route会再使用吗？/connection回重复使用吗？)，由此开始重试流程。





