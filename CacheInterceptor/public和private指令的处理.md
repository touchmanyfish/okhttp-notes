# public和private指令的处理

根据rfc 7234，当private cache收到的response中有这2个指令的时候可以(MAY)无视其他约束然后缓存这个response。

## OkHttp中的处理
```kotlin
// CacheStrategy.isCacheable()

        HTTP_MOVED_TEMP,
        StatusLine.HTTP_TEMP_REDIRECT -> {
          // These codes can only be cached with the right response headers.
          // http://tools.ietf.org/html/rfc7234#section-3
          // s-maxage is not checked because OkHttp is a private cache that should ignore s-maxage.
          if (response.header("Expires") == null &&
              response.cacheControl.maxAgeSeconds == -1 &&
              !response.cacheControl.isPublic &&
              !response.cacheControl.isPrivate) {
            return false
          }
        }
```

只在判断HTTP_MOVED_TEMP和StatusLine.HTTP_TEMP_REDIRECT是否cacheable时使用了这2个值。