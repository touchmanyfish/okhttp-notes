# Duplex connections are not supported for HTTP_1报错分析

## 报错分析

在标准HTTP实现中，只有HTTP/2和HTTP3/支持duplex。

在OkHttp中可以通过RequestBody#isDuplex()返回true来声明这个这个本次请求期望使用双工机制。然而OkHttp在为请求寻找connection时并不会读取
call.requestBody.isDuplex()。所以可能为这个期望使用双工机制的请求捞出一个HTTP/1.1的connnection。显然后续流程没法继续下去了。

OkHttp会在Http1ExchangeCodec#createRequestBody()检测这种非法的情况，并抛出异常。
```kotlin
override fun createRequestBody(request: Request, contentLength: Long): Sink {
    return when {
      // gg!  
      request.body != null && request.body.isDuplex() -> throw ProtocolException(
          "Duplex connections are not supported for HTTP/1")
      request.isChunked -> newChunkedSink() // Stream a request body of unknown length.
      contentLength != -1L -> newKnownLengthSink() // Stream a request body of a known length.
      else -> // Stream a request body of a known length.
        throw IllegalStateException(
            "Cannot stream a request body without chunked encoding or a known content length!")
    }
  }
```

OkHttp也不会为ProtocolException发起重试。

### 举例

```kotlin
private fun duplex1_1httpDemo(){
    val okHttpClient = OkHttpClient
        .Builder()
        // 让连接池中只有1.1的connection
        // 这里为了举例这么配置
        .protocols(listOf(Protocol.HTTP_1_1))
        .build()

    val request: Request = Request.Builder()
        .url("http://127.0.0.1:8081/userinfopost")
        .post(
            // body.isDuplex() == true
            body = TextStringRequestBody("haha")
        )
        .build()

    // 期望duplex，但是池子里只能捞到1.1的connection
    // 报错ProtocolException
    val result = okHttpClient.newCall(request).execute()
}
```


## 未理解的问题(TODO)

为什么在为请求捞取connection时不判断isDuplex()从而避免这种非法的情况出现？
