# cache的store流程

## rfc中关于“response是否能store的条件”
[rfc 7234 3.  Storing Responses in Caches](https://www.rfc-editor.org/rfc/rfc7234#section-3)

cache不能store response 除非：
* A:cache理解这个method，并且method此时满足具备cacheable的条件(defined as being
  cacheable),and
* B:cache理解status code,and
* C:request/response中都不包含"no-store"指令,or
* E:如果cache为shared，response中不包含“private”指令，and
* F:not shared cache || (no Authorization header && shared cache) || (Authorization header && shared cache && (public or s-max-age or must-revalidate))，and
* G:either:
  * 1.contains an Expires header field,or
  * 2.contains a max-age response directive,or
  * 3.contains a s-maxage response directive (see Section 5.2.2.9)
    and the cache is shared,or
  * 4.contains a Cache Control Extension (see Section 5.2.3) that
    allows it to be cached, or
  * 5.has a status code that is defined as cacheable by default (see
    Section 4.2.2), or
  * 6.contains a public response directive (see Section 5.2.2.5).

 * H:cache-control extension可以覆盖上述所有要求

### 定义特点

rfc只定义了“满足什么条件才能存”，并没有定义“什么时候存”

## OkHttp中实现

```kotlin
// CacheInterceptor#intercept(chain: Interceptor.Chain)

if (response.promisesBody() && CacheStrategy.isCacheable(response, networkRequest)) {
  // Offer this request to the cache.
  // metadata写入缓存  
  val cacheRequest = cache.put(response)
  // response写入缓存
  return cacheWritingResponse(cacheRequest, response).also {
    if (cacheResponse != null) {
      // This will log a conditional cache miss only.
      listener.cacheMiss(call)
    }
  }
}

// Cache#put(response: Response)

internal fun put(response: Response): CacheRequest? {
  val requestMethod = response.request.method

  if (HttpMethod.invalidatesCache(response.request.method)) {
    try {
      remove(response.request)
    } catch (_: IOException) {
      // The cache cannot be written.
    }
    return null
  }

  if (requestMethod != "GET") {
    // Don't cache non-GET responses. We're technically allowed to cache HEAD requests and some
    // POST requests, but the complexity of doing so is high and the benefit is low.
    return null
  }

  if (response.hasVaryAll()) {
    return null
  }

  //  这里将metadatq写入缓存
  val entry = Entry(response)
  var editor: DiskLruCache.Editor? = null
  try {
    editor = cache.edit(key(response.request.url)) ?: return null
    entry.writeTo(editor)
    return RealCacheRequest(editor)
  } catch (_: IOException) {
    abortQuietly(editor)
    return null
  }
}
```
上述就是OkHttp中store流程的相关代码。

### store流程执行的时机

当如下情况时会执行store流程代码：

* OkHttp手动构造了validation request GET，但是收到了非304的response
* OkHttp没有复用缓存，选择了直接发起请求

### 是否能store的判断

OkHttp对于是否能store的判断分散在了:
* response.promisesBody()
* CacheStrategy.isCacheable(response, networkRequest)
* cache.put(response)中执行缓存操作前有判断

让我们对比着rfc中定义的“是否能store”来看OkHttp忽略了哪些条件/没有实现哪些条件/实现了哪些条件/自己新增了哪些条件

#### 忽略的条件

**shared cahce相关条件:**

由于OkHttp的cache为private的，所以它忽略了rfc中定义的和shared cache相关的要求：

* E
* F
* G-3

#### 没有实现的条件

G条件中除了G-5都未实现，这收紧了缓存条件，目前不知道为何(**OKHTTP_TODO**)

#### 实现的条件

A:
OkHttp只会缓存:request(GET) - response
```kotlin
// Cache#put(response)

if (requestMethod != "GET") {
  // Don't cache non-GET responses. We're technically allowed to cache HEAD requests and some
  // POST requests, but the complexity of doing so is high and the benefit is low.
  return null
}
```

**B/G-5:**

OkHttp认为206不是cacheable；除此之外，和rfc定义一致。判定在CacheStrategy#isCacheable()中进行。

**C:**

和rfc定义一致。判定在CacheStrategy#isCacheable()中进行。

#### 新增的条件

**需要可能具有body:**

使用Response.promisesBody()进行判定；rfc中并未明确规定不能缓存没有body的response。

**Varys:*:**

当判定response是否能reuse时，需要判定varys match;根据定义“*”表示总是match失败。OkHttp选择了不缓存这种总是match失败的response；虽然rfc未明确定义
是否能缓存这种response。

```kotlin
//Cache#put(response)

if (response.hasVaryAll()) {
  return null
}

// Cache
fun Response.hasVaryAll() = "*" in headers.varyFields()
```

### 实际的写入操作

OkHttp缓存request(metadata)-response时分为为2步：

1. 缓存metadata
2. 缓存response 

但是不知道(**OKHTTP_TODO**):

1. 为什么要分2步
2. 每一步实现有如何特点

**缓存metadata：** 

```kotlin
// Cache#put(response)

val entry = Entry(response)
var editor: DiskLruCache.Editor? = null
try {
    editor = cache.edit(key(response.request.url)) ?: return null
    entry.writeTo(editor)
    return RealCacheRequest(editor)
} catch (_: IOException) {
    abortQuietly(editor)
    return null
}
```

**缓存response:**

```kotlin
return cacheWritingResponse(cacheRequest, response).also {
    if (cacheResponse != null) {
        // This will log a conditional cache miss only.
        listener.cacheMiss(call)
    }
}
```