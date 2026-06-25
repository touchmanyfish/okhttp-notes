# computeFreshnessLifetime()

```kotlin
    /**
 * Returns the number of milliseconds that the response was fresh for, starting from the served
 * date.
 */
private fun computeFreshnessLifetime(): Long {
    // A:有max-age
    val responseCaching = cacheResponse!!.cacheControl
    if (responseCaching.maxAgeSeconds != -1) {
        return SECONDS.toMillis(responseCaching.maxAgeSeconds.toLong())
    }

    //B:有Expires header
    val expires = this.expires
    if (expires != null) {
        val servedMillis = servedDate?.time ?: receivedResponseMillis
        val delta = expires.time - servedMillis
        return if (delta > 0L) delta else 0L
    }

    // C:有Last-Modifed header,利用它计算一个启发值
    if (lastModified != null && cacheResponse.request.url.query == null) {
        // As recommended by the HTTP RFC and implemented in Firefox, the max age of a document
        // should be defaulted to 10% of the document's age at the time it was served. Default
        // expiration dates aren't used for URIs containing a query.
        val servedMillis = servedDate?.time ?: sentRequestMillis
        val delta = servedMillis - lastModified!!.time
        return if (delta > 0L) delta / 10 else 0L
    }

    // D:
    return 0L
}
```
这个方法用于计算[rfc 7234 4.2.1.  Calculating Freshness Lifetime](https://www.rfc-editor.org/rfc/rfc7234#section-4.2.1)中
定义的Freshness Lifetime。

**s-maxage：**

这里的cache是private cache，于是OkHttp就直接不处理s-maxage。

**B:有Expires header:**

当Date header不存在时有如下关系图
```text
 Date(如果存在时对应的时刻)
   |------------------->|
               receivedResponseMillis
```
此时虽然expires.time - receivedResponseMillis会得出一个相比于Date header存在时，更小的Freshness Lifetime。
```text
|-----------------------------Expires - Date--------------|
|------Expires - receivedResponseMillis------|
```
显然，response如果在更小的Freshness Lifetime里新鲜，必然在更大的Freshness Lifetime也新鲜，只是会更早的stale罢了。所以这么计算也是有意义的。

如何理解结果为0的值呢？(**OKHTTP_TODO**)

**C:有Last-Modifed header,利用它计算一个启发值**

当Date header不存在时有如下关系图
```text
sentRequestMillis          Date(如果存在时对应的时刻)
       |------------------->|--------------------->|
                                          receivedResponseMillis  
```
使用sentRequestMillis参与计算会得出一个更小的Freshness Lifetime，根据上一小节的讨论这种更小的值也是有意义的。如果使用receivedResponseMillis
则会得出一个更大的值，这可能导致新鲜度判断不准确,例如：
```text
// 根据rfc 7234中的定义，age此时已经是stale了，但是却被错误的判定为fresh

Date(如果存在时) - lastModified < age < receivedResponseMillis - lastModified
```

如何理解结果为0的值呢？(**OKHTTP_TODO**)

```kotlin
if (lastModified != null && cacheResponse.request.url.query == null) {
    //...
}
```
OkHttp只会缓存GET方法，所以上述代码表示当GET方法不带"?"时，才能使用lastModified计算Freshness Lifetime。这实际是在实现:
```text
 Note: Section 13.9 of [RFC2616] prohibited caches from calculating
      heuristic freshness for URIs with query components (i.e., those
      containing '?').  In practice, this has not been widely
      implemented.  Therefore, origin servers are encouraged to send
      explicit directives (e.g., Cache-Control: no-cache) if they wish
      to preclude caching.
```

**D:**

如何理解返回0？(**OKHTTP_TODO**)

