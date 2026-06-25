# cache的compute()流程

```kotlin
// CacheStrategy.kt

/** Returns a strategy to satisfy [request] using [cacheResponse]. */
fun compute(): CacheStrategy {
    val candidate = computeCandidate()

    // We're forbidden from using the network and the cache is insufficient.
    if (candidate.networkRequest != null && request.cacheControl.onlyIfCached) {
        return CacheStrategy(null, null)
    }

    return candidate
}
```
compute()方法返回如下4种结果:
```
// A
return CacheStrategy(request, null)

// B
return CacheStrategy(null, builder.build())

// C
return CacheStrategy(conditionalRequest, cacheResponse)

// D
return CacheStrategy(null, null)
```

**A:**

compute()判定此时没法进行复用，接下来需要发起网络请求；

**B:**

compute()判定此时有可复用缓存，接下来不用发起网络请求，直接复用复用即可；

**C:**

compute()判定此时需要发起条件验证，于是根据response中的validator构建一个conditionalRequest；接下来需要讲这个请求发往服务器。

**D:**

compute()判定：request中有only-if-cache但是本地目前没有可复用缓存;后续需要返回504给client；


compute()的作用就是参考rfc中的定义，在计算下一步是干什么：
```
(request,cache) -> (network || conditional request || reuse || 504)
```

## 调用时机

1. OkHttp会先在本地查找满足的cacheCandidate：
   * URI match
   * method match
   * varys match

2. 让cacheCandidate参与构造CacheStrategy,然后调用compute()方法。
```
## computeCandidate()


