# Cache.kt

## get(request)方法

```kotlin
  internal fun get(request: Request): Response? {
    val key = key(request.url)
    val snapshot: DiskLruCache.Snapshot = try {
        cache[key] ?: return null
    } catch (_: IOException) {
        return null // Give up because the cache cannot be read.
    }

    val entry: Entry = try {
        Entry(snapshot.getSource(ENTRY_METADATA))
    } catch (_: IOException) {
        snapshot.closeQuietly()
        return null
    }

    // A
    val response = entry.response(snapshot)
    // B
    if (!entry.matches(request, response)) {
        response.body?.closeQuietly()
        return null
    }

    return response
}
```

缓存的存储形式为：

```text
key(url) - snapshot
```

get(request)会获取key(request.url)对应的snapshot，从其构建一个response。然后判断response是否满足部分reuse条件：

1. uri match
2. method match
3. varys match

response只是作为候选人，是否能reuse还需要进一步流程。

### 和rfc 7234中实现有出入的地方

#### 1.method match的实现

```kotlin
// entry.matches(request, response)
fun matches(request: Request, response: Response): Boolean {
    return url == request.url &&
            requestMethod == request.method &&
            varyMatches(response, varyHeaders, request)
}
```

matches()方法实现了“候选人”判断。

在[rfc 7234 4. Constructing Responses from Caches](https://www.rfc-editor.org/rfc/rfc7234#section-4)
中判断reuse时对于method的规定为：

```text
   o  the request method associated with the stored response allows it
      to be used for the presented request, and
```

而OkHttp则直接处理为

```text
requestMethod == request.method
```

#### 2.primary key/secondary key

[rfc 7234 2. Overview of Cache Operation](https://www.rfc-editor.org/rfc/rfc7234#section-2)

```text
   The primary cache key consists of the request method and target URI.
   However, since HTTP caches in common use today are typically limited
   to caching responses to GET, many caches simply decline other methods
   and use only the URI as the primary cache key.

   If a request target is subject to content negotiation, its cache
   entry might consist of multiple stored responses, each differentiated
   by a secondary key for the values of the original request's selecting
   header fields (Section 4.1).
```

最开始我理解cache中的存储方式为

```text
// 1.
key(uri+method+varys) - response

// 2.
key(uri+varys) - response
```

而OkHttp却是

```text
key(url) - snapshot

fun key(url: HttpUrl): String = url.toString().encodeUtf8().md5().hex()
```

method和varys参与后续判断

### 不懂的地方

// B处是在进行缓存是否可以满足复用的部分条件的判断：

1. uri match
2. method match
3. vary match

在// A执行之前的entry中已经包含了对比需要的上述信息，所以这个判断在//
A之前不是更好？因为如果判断不通过则可以避免了entry.response(snapshot)
的执行，性能不就更好了？

## put()方法

### okHttp只会存GET方法
```kotlin
//Cache#put()

if (requestMethod != "GET") {
    // Don't cache non-GET responses. We're technically allowed to cache HEAD requests and some
    // POST requests, but the complexity of doing so is high and the benefit is low.
    return null
}
```