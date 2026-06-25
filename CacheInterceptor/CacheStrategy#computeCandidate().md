# CacheStrategy#computeCandidate()

computeCandidate()主要用于计算如下三种情况

```text
// A
return CacheStrategy(request, null)

// B
return CacheStrategy(null, builder.build())

// C
return CacheStrategy(conditionalRequest, cacheResponse)
```

如果收到的request中携带来only-if-cached，那么在CacheStrategy#compute()中，A/C会变成D

```text
// D
return CacheStrategy(null, null)
```

## computeCandidate()内部执行流程

1. 判定A(筛选只能发起网络请求的情况)
2. 判定B(筛选可以复用的情况)
3. 判定C(剩下的就帮它发起网络请求，除非没法生成validator)

### 判定A

```kotlin
// 1
// No cached response.
if (cacheResponse == null) {
    return CacheStrategy(request, null)
}

// 2
// Drop the cached response if it's missing a required handshake.
if (request.isHttps && cacheResponse.handshake == null) {
    return CacheStrategy(request, null)
}

// 3
// If this response shouldn't have been stored, it should never be used as a response source.
// This check should be redundant as long as the persistence store is well-behaved and the
// rules are constant.
if (!isCacheable(cacheResponse, request)) {
    return CacheStrategy(request, null)
}

val requestCaching = request.cacheControl
if (
// 4
    requestCaching.noCache
    // 5
    || hasConditions(request)
) {
    return CacheStrategy(request, null)
}
```

#### A-1

cacheReponse不为空时满足：

* URI match
* method match
* varys match

这里显然就是在检查是否有满足reuse的基本条件([rfc 7234 4. Constructing Responses from Caches
](https://www.rfc-editor.org/rfc/rfc7234.html#section-4))

#### A-2

OKHTTP_TODO

#### A-3

```kotlin
fun isCacheable(response: Response, request: Request): Boolean {
    //...其他代码
    return !response.cacheControl.noStore && !request.cacheControl.noStore
}
```

因为在执行store操作的时候也会调用isCacheable()，所以当执行到A-4的时候isCacheable()方法等同于:

```kotlin
fun isCacheable(response: Response, request: Request): Boolean {
    return !request.cacheControl.noStore
}
```

这表示当OkHttp收到request(no-store)时会直接发起网络请求而不会继续判定是否能reuse。这样实现并 **没有违反rfc 7234的定义**
；因为其中并没有对“当cache
收到了request(no-store)时应该怎么做”进行定义。

gemini说OkHttp这么实现是因为当收OKHttp到request(no-store)时认为(**OKHTTP_TODO**)：

* 用户不想缓存对应的response
* 用户不想结果被缓存干扰

#### A-4

```text
5.2.1.4.  no-cache

   The "no-cache" request directive indicates that a cache MUST NOT use
   a stored response to satisfy the request without successful
   validation on the origin server.
```

[rfc 72345.2.1.4. no-cache](https://www.rfc-editor.org/rfc/rfc7234#section-5.2.1.5) 中定义，当cache收到request(no-cache)
时不能进行resuse流程，除非先对对应stored response进行validation。我目前将“validation”理解为
“为这个stored response构造一个conditional request，发往服务器进行验证”。

但是OkHttp遇到这种情况却选择跳过接下来的reuse判断，然后直接将收到的request发往下一跳；

```kotlin
// rfc定义
reuse      -> stored response
validation -> 1. 304 -> 更新header -> 返回stored response
2.完整response -> 删除缓存/保存最新的 -> 返回stored response

// okhttp中实现
直接转发request到下一跳 -> store流程 -> 返回收到的response
```

**OkHttp实现no-store request指令的原因猜测(OKHTTP_TODO):**

* 与rfc定义相比，OkHttp的实现需要处理的分支变少了，并且可能会消耗更多的流量。但是这种实现增加了no-store
  request指令在OkHttp中可预测性增加；
  同时简化了实现。
* gemini说开发中在request使用no-store指令的时候，期望cache不要让复用逻辑来干扰对no-store指令结果的预测

#### A-5

OKHttp对于conditional requst的处理如下：

1. OkHttp发现有If-Modified-Since/If-None-Match时就会明确的将request转发到下一跳
2. 其他conditional headers一律当不存在；
3. OkHttp不会缓存response(206)

显然第二条会导致client收到的结果和预期不符并且出现一些问题。

```kotlin
private fun hasConditions(request: Request): Boolean =
    request.header("If-Modified-Since") != null || request.header("If-None-Match") != null
```

**“If-Modified-Since/If-None-Match”：**

虽然 [rfc 7234 4.3.2. Handling a Received Validation Request](https://www.rfc-editor.org/rfc/rfc7234#section-4.3.2)
中允许cache可以处理conditional header，但是当OkHttp遇到这2个conditional header时，选择将request直接转发。

gemini说此时OkHttp认为(**OKHTTP_TODO**):

* 虽然rfc定义了如何处理但是这增加了实现的复杂度
* 同时实现会降低结果的准确性

**If-Match/If-Ummodified-Since:**

OkHttp没有对这2个conditional header进行判断;当cache收到request(If-Match)这导致了一些有趣的情况：

1. cache中已经有request-response
2. cache收到request(If-Match)
3. computeCandidate()中reuse逻辑判断通过，cache选择将stored response返回给client；或者
4. reuse()逻辑未通过，OkHttp根据stored response的validator尝试手动构建条件请求；或者
5. OkHttp构建条件请求失败，最终选择直接将request转发到下一跳。

当client发起request(If-Match)时，期望去orignal server验证。但是在OkHttp当实现中，只有5.才能实现client的期望。

为啥OkHttp在遇到conditional headers的时候不直接转发request到下一跳，非要这么实现呢，这不就降低了可预测性？目前难以理解(*
*OKHTTP_TODO**)。

**Range:**

OkHttp默认不缓存response(206)。

当本地有request-response时：

1. cache收到request(Range)
2. computeCandidate()中reuse逻辑判断通过，cache选择将完整的stored response(code)返回给client；或者
3. reuse()逻辑未通过，OkHttp根据stored response的validator尝试手动构建条件请求；或者
4. OkHttp构建条件请求失败，最终选择直接将request转发到下一跳。

client期望:

* 收到206/416/其他 response code(RFC_GUESS)
* 要求的范围

当reuse逻辑通过时：

* 返回完整的store response(cacheable code)

**If-Range/Range:**
当本地有request-response(v1)时：

1. cache收到request(Range,If-Range(v2))
2. computeCandidate()中reuse逻辑判断通过，cache选择将完整的stored response(v1)返回给client；或者
3. reuse()逻辑未通过，OkHttp根据stored response的validator(v2)尝试手动构建条件请求；或者
4. OkHttp构建条件请求失败，最终选择直接将request转发到下一跳。

当resuse()逻辑通过时，If-Range(v2)
期望V2版本，但是OkHttp却返回了v1版本给client。这是个明显的错误。不知道为什么OkHttp要这么实现(OKHTTP_TODO)

## 判定B

```kotlin
 val responseCaching = cacheResponse.cacheControl

// 1
val ageMillis = cacheResponseAge()
//2
var freshMillis = computeFreshnessLifetime()

if (requestCaching.maxAgeSeconds != -1) {
    freshMillis = minOf(freshMillis, SECONDS.toMillis(requestCaching.maxAgeSeconds.toLong()))
}

var minFreshMillis: Long = 0
if (requestCaching.minFreshSeconds != -1) {
    minFreshMillis = SECONDS.toMillis(requestCaching.minFreshSeconds.toLong())
}

var maxStaleMillis: Long = 0
if (
// 3
    !responseCaching.mustRevalidate
    && requestCaching.maxStaleSeconds != -1
) {
    maxStaleMillis = SECONDS.toMillis(requestCaching.maxStaleSeconds.toLong())
}

if (
// 4
    !responseCaching.noCache
    &&
    // 5
    ageMillis + minFreshMillis < freshMillis + maxStaleMillis
) {
    val builder = cacheResponse.newBuilder()
    // 6
    if (ageMillis + minFreshMillis >= freshMillis) {
        builder.addHeader("Warning", "110 HttpURLConnection \"Response is stale\"")
    }
  
    // 7
    val oneDayMillis = 24 * 60 * 60 * 1000L
    if (ageMillis > oneDayMillis && isFreshnessLifetimeHeuristic()) {
        builder.addHeader("Warning", "113 HttpURLConnection \"Heuristic expiration\"")
    }
    return CacheStrategy(null, builder.build())
}
```

### B-1

参考[CacheStrategy#cacheResponseAge()](CacheStrategy%23cacheResponseAge().md)

### B-2

参考[CacheStrategy#computeFreshnessLifetime()](CacheStrategy#computeFreshnessLifetime())

### B-3

must-revalidate表示如果不能给client提供stale的副本，除非对其进行了验证([rfc 7234 5.2.2.1. must-revalidate](https://www.rfc-editor.org/rfc/rfc7234#section-5.2.2.1))。
max-stale明显违反must-revalidate的语义，所以OkHttp在遇到must-revalidate时选择不计算max-stale

### B-4

[rfc 7234 5.2.2.2. no-cache](https://www.rfc-editor.org/rfc/rfc7234#section-5.2.2.1) 中response(no-cache)处理分为带值和不带
值的情况，OkHttp则统统按照不带值进行处理：如果response中有no-cache指令，则直接进入“通过validator构造条件请求”逻辑。

### B-5

B除代码用于判断是否可以reuse cacheResponse；

**ageMillis + minFreshMillis < freshMillis + maxStaleMillis公式的推导：**

思路是依次使用Cache-Control对lifetime进行约束计算出新的adjustedLifetime。当相关约束都进行了计算以后，将最终的adjustedLifetime带入
[rfc 7234 4.2. Freshness](https://www.rfc-editor.org/rfc/rfc7234.html#section-4.2) 中定义的新鲜度判定公式中

```text
response_is_fresh = (freshness_lifetime > current_age)
```

可以得出OkHttp代码中使用的判断公式

```text
age/freshness lifetime

// rfc中定义的新鲜度判定公式
current_age < lifetime

// 满足max-age
current_age <= max-age
adjustedLifetime = min(lifetime,max-age)

// 满足max-stale
current_age <= lifetime + max-stale
adjustedLifetime = min(lifetime,max-age) + max-stale

// 满足 min-fresh
current_age <= lifetime - min-fresh
adjustedLifetime = min(lifetime,max-age) + max-stale - min-fresh

// 变形
ageMillis + min-fresh < min(lifetime,maxage) + max-stale
// 最终得出代码中相同判断
ageMillis + minFreshMillis < freshMillis + maxStaleMillis
```

**目前的理解(OKHTTP_TODO)：**

* 上述推导是我猜的
* ageMillis + minFreshMillis < freshMillis + maxStaleMillis：OkHttp这样实现有啥问题目前还没头绪


### B-6/B-7 

这两块代码就有缘在来搞懂吧。。

## C

```kotlin
 // Find a condition to add to the request. If the condition is satisfied, the response body
// will not be transmitted.
val conditionName: String
val conditionValue: String?
when {
    etag != null -> {
        conditionName = "If-None-Match"
        conditionValue = etag
    }

    lastModified != null -> {
        conditionName = "If-Modified-Since"
        conditionValue = lastModifiedString
    }

    servedDate != null -> {
        conditionName = "If-Modified-Since"
        conditionValue = servedDateString
    }

    else -> return CacheStrategy(request, null) // No condition! Make a regular request.
}

val conditionalRequestHeaders = request.headers.newBuilder()
conditionalRequestHeaders.addLenient(conditionName, conditionValue!!)

val conditionalRequest = request.newBuilder()
    .headers(conditionalRequestHeaders.build())
    .build()
return CacheStrategy(conditionalRequest, cacheResponse)
```

这部分代码用于根据当A/B筛选都没通过时，根据cacheResponse的validator帮它手动构建一个conditional request。