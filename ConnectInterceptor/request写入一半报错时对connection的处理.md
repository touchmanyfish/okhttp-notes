# request写入一半报错时对connection的处理


## http1.1

除了抛弃这个connection别无它法，OkHttp也是这么处理的。这里没什么好分析的。


## http2.0

### 结论

1. OkHttp总是会尝试向对面发送一个RST_STREAM
2. OkHttp根据情况会将connection标记为不可复用

### RetryAndFollowUpInterceptor的finally块realCall.exhange还未解绑

当realCall的对应exhange创建的requestBody/responseBody都被关闭时，会与exhange解绑。而如果在写入request阶段就报错，执行到
RetryAndFollowUpInterceptor的finally块，此时responseBody还未被创建，显然此时exhange还未被解绑。

### 分析:结论1

根据上一小节的结论,当写入request阶段报错，如下方法总是被执行

```kotlin
// RetryAndFollowUpInterceptor的finall块
// ->call.exitNetworkInterceptorExchange(closeActiveExchange)
// ->exchange?.detachWithViolence()
fun detachWithViolence() {
    // 尝试向服务器发送RST_STREAM
    codec.cancel()
    call.messageDone(this, requestDone = true, responseDone = true, e = null)
}
```

### 分析:结论2

当写入request.body阶段报错时还会进入到trackFailure()方法。

```kotlin
// Exchange.kt
private fun trackFailure(e: IOException) {
    hasFailure = true
    finder.trackFailure(e)
    codec.connection.trackFailure(call, e)
}

// codec.connection.trackFailure(call, e)
@Synchronized
internal fun trackFailure(call: RealCall, e: IOException?) {
    if (e is StreamResetException) {
        when {
            e.errorCode == ErrorCode.REFUSED_STREAM -> {
                // Stop using this connection on the 2nd REFUSED_STREAM error.
                refusedStreamCount++
                if (refusedStreamCount > 1) {
                    noNewExchanges = true
                    routeFailureCount++
                }
            }

            e.errorCode == ErrorCode.CANCEL && call.isCanceled() -> {
                // Permit any number of CANCEL errors on locally-canceled calls.
            }

            else -> {
                // Everything else wants a fresh connection.
                noNewExchanges = true
                routeFailureCount++
            }
        }
    } else if (!isMultiplexed || e is ConnectionShutdownException) {
        noNewExchanges = true

        // If this route hasn't completed a call, avoid it for new connections.
        if (successCount == 0) {
            if (e != null) {
                connectFailed(call.client, route, e)
            }
            routeFailureCount++
        }
    }
}
```

观察RealConnection#trackFailure()方法，你不用弄明白每个分支具体对应的场景，就能得出在这个方法中:

1. 当使用HTTP/2.0 的connection时所有分支都有进入的可能
2. 只有部分分支会将connection标记为不可复用(noNewExchanges == true)

结合本小节的特定场景我们有如下结论:

当在HTTP/2.0 的connection上写入request.body报错时，OkHttp可能会将connection标记为不可复用。
