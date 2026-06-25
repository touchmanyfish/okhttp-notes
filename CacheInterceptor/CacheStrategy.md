# CacheStrategy.kt


## OkHttp不处理收到的条件请求的情况
后续收到response以后，如何缓存？（**OKHTTP_TODO**）
```kotlin
// computeCandidate()
val requestCaching = request.cacheControl
if (requestCaching.noCache || hasConditions(request)) {
    // 这里表示不处理，因为第二个参数cacheResponse为null
    return CacheStrategy(request, null)
}

/**
 * Returns true if the request contains conditions that save the server from sending a response
 * that the client has locally. When a request is enqueued with its own conditions, the built-in
 * response cache won't be used.
 */
private fun hasConditions(request: Request): Boolean =
    request.header("If-Modified-Since") != null || request.header("If-None-Match") != null

5.2.1.4.  no-cache

The "no-cache" request directive indicates that a cache MUST NOT use
a stored response to satisfy the request without successful
validation on the origin server.

```

那么其他几个If-x呢？如何处理？



