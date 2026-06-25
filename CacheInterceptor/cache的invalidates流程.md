# cache的invalidation流程

rfc 7234规定method **会参与** 否进行invalidates判断；OkHttp **只** 使用HttpMethod#invalidatesCache()
判断是否执行invalidates操作，其中 **只会**
使用method进行判断。

## rfc中invalidate相关定义

[rfc 7234 4.4. Invalidation](https://www.rfc-editor.org/rfc/rfc7234#section-4.4)

### invalidates操作的条件

收到的response如果满足如下条件则必须(**MUST**)执行invalidates操作:

* method为unsafe/unknow method
* code 为2xx或者3xx

### invalidates范围

* effective Request URI相关的stored responses
* Location/Content-Location包含的URI相关的stored对应的responses

当Location/Content-Location包含的URI的host部分和effective Request URI包含的host部分不相等时，禁止(**MUST NOT**)
执行invalidates操作。

### 当判定为invalidates时，实际执行的操作

* 删除所有effective request URI对应的缓存，或者
* 把它们标记为"invalid"

## rfc 7231中定义的safe method

[rfc 7231 4.2.1. Safe Methods](https://www.rfc-editor.org/rfc/rfc7231.html#section-4.2.1)

```text
GET
HEAD
OPTIONS
TRACE
```

显然，其他的method就是unsafe method。

## OkHttp中的实现

术语定义:
* invalidates流程 = 是否能invalidates操作+invalidates实际执行内容

当如下情况时：

* OkHttp手动构造了validation request GET，但是收到了非304的response
* OkHttp没有复用缓存，选择了直接发起请求,收到了response

会执行invalidates代码块。
```kotlin
//invalidates操作对应的代码块

// 只使用invalidatesCache()进行判断
if (HttpMethod.invalidatesCache(networkRequest.method)) {
    try {
        // 实际的删除操作
        cache.remove(networkRequest)
    } catch (_: IOException) {
        // The cache cannot be written.
    }
}
```

### invalidates代码块的执行时机

```kotlin
// CacheInterceptor#intercept(chain: Interceptor.Chain)

if (cache != null) {
    if (response.promisesBody() && CacheStrategy.isCacheable(response, networkRequest)) {
        // Offer this request to the cache.
        // A:put()方法中也会执行是否invalidates判断
        val cacheRequest = cache.put(response)
        return cacheWritingResponse(cacheRequest, response).also {
            if (cacheResponse != null) {
                // This will log a conditional cache miss only.
                listener.cacheMiss(call)
            }
        }
    }

    // B:执行是否invalidates判断   
    if (HttpMethod.invalidatesCache(networkRequest.method)) {
        try {
            // C:执行删除操作
            cache.remove(networkRequest)
        } catch (_: IOException) {
            // The cache cannot be written.
        }
    }
```

当如下情况时会执行上述代码：

* OkHttp手动构造了validation request GET，但是收到了非304的response
* OkHttp没有复用缓存，选择了直接发起请求

另外观察A/B除代码发现：

* A处cache.put(response)中和B处会执行相似的代码块

```kotlin
if (HttpMethod.invalidatesCache(networkRequest.method)) {
    try {
        cache.remove(networkRequest)
    } catch (_: IOException) {
        // The cache cannot be written.
    }
}
```

显然这部分代码块的执行时机与上述代码块中的if无关，只取决于前面提到的2种情况。此时推测其为OkHttp实现的invidates流程，为了方便，将其命名为“invalidates代码块”。
通过invalidates代码块可以看出:

* Okhttp中只使用invalidatesCache()来判断是否执行实际的invalidates操作
* 实际的invalidates操作为cache.remove(networkRequest)

rfc中只定义了“判断是否能执行invalidates+invalidates如何执行”， 并没有定义"什么时候可以进行invalidates流程"。

### invalidatesCache()实现

```kotlin
// HttpMethod.kt

fun invalidatesCache(method: String): Boolean = (
        method == "POST" ||
                method == "PATCH" ||
                method == "PUT" ||
                method == "DELETE" ||
                method == "MOVE") // WebDAV
```

**关于method是否满足执行invalidation对method对要求：**

| 特性 / 方法           | RFC 7234 规定         | OkHttp 实现                       |
|:------------------|:--------------------|:--------------------------------|
| **unsafe method** | 只要是unsafe method就满足 | 当POST/PATCH/PUT/DELETE/MOVE时才满足 |
| **unknow method** | 只要是unknow method就满足 | 不满足                             |

对于"判定是否执行invalidates操作",OkHttp只使用了部分unsafe方法进行判断，仅此而已。

### 为什么这样实现？

OKHTTP_TODO

### invalidates时，实际执行的操作
```kotlin
@Throws(IOException::class)
internal fun remove(request: Request) { 
    cache.remove(key(request.url))
}
```
OkHttp直接把key(request.url)对应的缓存都给删了，根本没管Location/Content-Location部分。





