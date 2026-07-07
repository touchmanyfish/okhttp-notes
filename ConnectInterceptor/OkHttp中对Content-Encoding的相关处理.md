# OkHttp中对Content-Encoding的相关处理

## 处理逻辑

1. 请求中没有"Accept-Encoding"和"Range"时，OkHttp自动加上"Accept-Encoding:gzip"
2. 刚刚自动添加了gzip,如果响应包含"Content-Encoding: gzip"，则自动使用GzipSource进行解压

```kotlin
// BridgeInterceptor.kt

// If we add an "Accept-Encoding: gzip" header field we're responsible for also decompressing
// the transfer stream.
var transparentGzip = false
if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
    // 没使用Range&&没写Accept-Encoding，自动加上gzip
    transparentGzip = true
    requestBuilder.header("Accept-Encoding", "gzip")
}


if (transparentGzip &&
    "gzip".equals(networkResponse.header("Content-Encoding"), ignoreCase = true) &&
    networkResponse.promisesBody()) {
    
    // 收到请求了
    // 刚刚自动加了gzip && 响应编码为gzip
    
    val responseBody = networkResponse.body
    if (responseBody != null) {
        val gzipSource = GzipSource(responseBody.source())
        val strippedHeaders = networkResponse.headers.newBuilder()
            .removeAll("Content-Encoding")
            .removeAll("Content-Length")
            .build()
        responseBuilder.headers(strippedHeaders)
        val contentType = networkResponse.header("Content-Type")
        responseBuilder.body(RealResponseBody(contentType, -1L, gzipSource.buffer()))
    }
}
```

**总结一下：**

如果调用者手动加了"Accept-Encoding"，OkHttp则认为调用者想自己处理解码；如果没加则OkHttp自动加上gzip，并只处理"Content-Encoding: gzip"
这一种情况;其他情况则交由调用者处理。

```text
// OkHttp自动加上了gzip
// 但是服务器返回了
Content-Encoding: gzip,br

// OkHttp处理不了这种情况的解码，只好让调用层处理
```

### 为什么要判断一下"Range"?

```kotlin
var transparentGzip = false
if (userRequest.header("Accept-Encoding") == null 
    // 这里
    && userRequest.header("Range") == null) {
    // 没使用Range&&没写Accept-Encoding，自动加上gzip
    transparentGzip = true
    requestBuilder.header("Accept-Encoding", "gzip")
}
```

**不添擅自添加gzip的情况：**

```text
请求：
Range:x-y

服务器收到以后：
1.根据内容协商选定A版本
2.给客户端返回A版本的x-y部分
```

**擅自添加gzip的情况：**

```text
请求：
Range:x-y
Content-Encoding:gzip

服务器收到以后：
1.根据内容协商选定gzip版本
2.给客户端返回gzip版本的x-y部分
```

**问题：**

（A版本可能是gzip版本，这里我们假设A不是gzip版本的情况）
1. 由于擅自添加gzip，导致请求的目标发生变化，服务器返回的并不是请求期望的内容
2. 当擅自添加gzip以后，服务器的操作是“先压缩成gzip版本，然后返回其中x-y部分”，okhttp默认的GzipSource没法处理这种x-y。导致报错

## TODO

resource/representation/content-range/内容协商重构上一小节内容