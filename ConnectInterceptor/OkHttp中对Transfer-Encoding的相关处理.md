# OkHttp中对Transfer-Encoding的相关处理

## HTTP/2不支持Transfer-Encoding

后续讨论全部针对HTTP/1.1

## 对请求的header处理

```kotlin
// BridgeInterceptor.kt

val contentLength = body.contentLength()
if (contentLength != -1L) {
    // 能拿到长度就不使用分块传输
    requestBuilder.header("Content-Length", contentLength.toString())
    requestBuilder.removeHeader("Transfer-Encoding")
} else {
    // 拿不到长度则使用分块传输
    requestBuilder.header("Transfer-Encoding", "chunked")
    requestBuilder.removeHeader("Content-Length")
}
```

### 能拿到长度时

此时选择不使用分块传输，原因如下：

* 避免分块传输带来的额外开销(例如：1.每一块的末位会添加\r\n 2.进行分块会涉及内存拷贝)
* 提高兼容性：很多老旧服务器支持Content-Length但是不支持分块 


### 拿不到长度时

说明body对应动态内容，如果你想使用keep-alive特性同时不启用分块传输，你需要告知连接另一端本次传输的Content-Length,于是你需要将动态内容完整的
读到内存中，然后获取Content-Length。如果你的这个动态内容很大，会给你的内存造成巨大的负担。

这种情况下只能使用分块传输。

### 覆盖这个默认行为

实现覆盖行为的自定义拦截器，然后将其插入到networkInterceptors中。


## 同时存在 Content-Length 和 Transfer-Encoding 时

根据rfc定义，同时存在时:

1. 接收方必须忽略“Content-Length”
2. 如果接收方是proxy，需要删除“Content-Length”
3. 接收方可以选择返回 400 Bad Request 并关闭连接

OkHttp的按照其中第一条处理。

```kotlin
// Http1ExchangeCodec.kt

// 处理request时
override fun createRequestBody(request: Request, contentLength: Long): Sink {
    return when {
        request.body != null && request.body.isDuplex() -> throw ProtocolException(
            "Duplex connections are not supported for HTTP/1")
        // 优先使用Transfer-Encoding
        request.isChunked -> newChunkedSink() // Stream a request body of unknown length.
        contentLength != -1L -> newKnownLengthSink() // Stream a request body of a known length.
       
        else -> // Stream a request body of a known length.
            throw IllegalStateException(
                "Cannot stream a request body without chunked encoding or a known content length!")
    }
}

// 处理response时
override fun openResponseBodySource(response: Response): Source {
    return when {
        !response.promisesBody() -> newFixedLengthSource(0)
        // 优先使用Transfer-Encoding
        response.isChunked -> newChunkedSource(response.request.url)
        else -> {
            val contentLength = response.headersContentLength()
            if (contentLength != -1L) newFixedLengthSource(contentLength)
            else newUnknownLengthSource()
        }
    }
}
```

### 细节1

传递给createRequestBody()的contentLength并不是来自request的Content-Length header；而是来自body的真实大小。

### 细节2

```text
// 1.
Transfer-Encoding: chunked

// 2.
Transfer-Encoding: gzip, chunked
```

上述2种header都表示开启分块传输，但是Okhttp只判断第1种情况。

```kotlin
// 判断response中的header
private val Response.isChunked: Boolean
get() = "chunked".equals(header("Transfer-Encoding"), ignoreCase = true)

// 判断request中的header
private val Request.isChunked: Boolean
get() = "chunked".equals(header("Transfer-Encoding"), ignoreCase = true)
```


## 分块/Trailer

**ChunkedSink：** 将写入其中的数据进行分块

**ChunkedSource：** 从其中读取从分块中剥离出来的数据以及body尾部的Trailers

