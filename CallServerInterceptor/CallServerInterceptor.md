# CallServerInterceptor

## 处理1xx code

### 1xx在rfc中的相关定义

* [code 1xx](../rfc9110/15.2.%20Informational%201xx.md)
* [code 100](../rfc9110/15.2.1.%20100%20Continue.md)
* [code 102](../rfc/rfc%202518%2010.1%20102%20Processing.md)
* [code 103](../rfc/rfc%208297%202.%20%20HTTP%20Status%20Code%20103_Early%20Hints.md)

### 处理流程

**100-continue request:**

1. flush headers
2. 读取response首部
3. 如果code不为100则进入5.
4. 如果code为100则继续写入body；然后再次读取response首部；
5. 使用shouldIgnoreAndWaitForRealResponse()判断，如果为1xx(除了101)，则再次读取response首部

**其他resquest:**

1. flush headers
2. 读取response首部
3. 使用shouldIgnoreAndWaitForRealResponse()判断，如果为1xx(除了101)，则再次读取response首部

**101如何处理？:**

OKHTTP_TODO

### 连续收到1xx不会全部处理

rfc中定义，在同一个请求中，可能会收到多次1xx，client需要处理这种情况。

Okhttp只会尝试有限次数。

### 102的处理

服务器返回102表示服务器当前正在处理，client可能需要等一会再尝试。

OkHttp发现code为102时立即再读了一次，并没有做更多的特殊处理。

### 103的处理

服务器返回的103的首部中可能包含一些 **hints**，client拿到hints以后可以提前处理，以利用服务器处理请求时的时间间隙。

OkHttp发现code为103时立即再读了一次，并没有做更多的特殊处理。

## Duplex

http 2.0为Duplex。

Okhttp只会帮你flush request的首部。对于request body部分，需要开发中自己调用flush()或者close()来维护(例如push body数据到服务器)

## Half-Duplex下body写入分析

http 1.1为Half-Duplex。

**现在只讨论http 1.1**

```kotlin
//CallServerInterceptor#intercept(chain)

// Write the request body if the "Expect: 100-continue" expectation was met.
val bufferedRequestBody = exchange
    .createRequestBody(request, false)// 返回ChunkedSink/KnownLengthSink的包装
    .buffer()//用RealBufferedSink包裹调用者

// A
requestBody.writeTo(bufferedRequestBody)

// B
bufferedRequestBody.close()

//。。

if (requestBody == null || !requestBody.isDuplex()) {
    // C
    // 这里将A处写入到body push到远端。
    exchange.finishRequest()
}

```

当代码执行到A时有如下结构
```text
bufferedRequestBody(RealBufferedSink)
    -> RequestBodySink
    -> ChunkedSink/KnownLengthSink
    -> codec.sink
```

**RequestBodySink** 相当于一个delegate，会将调用转发到下游sink。

**ChunkedSink/KnownLengthSink：**

* 转发write()到下游
* 当close()方法被调用时并不会调用下游的close();也不会将当前Sink中数据写入到下游（它们根本就没有数据。

所以当bufferedRequestBody调用下游sink的write()方法时，数据会立即进入codec.sink中；

一般情况下sink链上游调用close()会导致下游所有的close()被调用。然而当bufferedRequestBody.close()被调用时，这种“连续调用”在ChunkedSink/KnownLengthSink
终止，codec.sink.close()并不会被调用到。所以bufferedRequestBody.close()并没有“将sink链上从当前位置开始到最后的数据push到server的效果”。

### A处代码

RealBufferedSink#write() 系列方法特点:

* 将数据写入自己的buffer
* 将写满的segments通过调用下游sink.write()写入下游sink

对于bufferedRequestBody那些写满的segments会立即被写入codec.sink.

在A处代码writeTo()方法中可以通过调用bufferedRequestBody.write()系方法来往其中写入数据;这些数据要么在bufferedRequestBody的buffer中，要么
在codec.sink中。

### B

RealBufferedSink#close()方法:

* 调用下游sink.write()将buffer中的数据全部写入下游sink
* 数据写完以后，调用下游sink.close()

所以调用bufferedRequestBody.close()以后buffer中的数据会立即进入codec.sink;另外根据前面的分析，这里的close()并没有“将sink链上从当前位
置开始到最后的数据push到server的效果”。

### C

C处代码最终会调用到codec.sink.flush(),用于将codec.sink中的数据push到服务器。

## request(100-continue)没有收到code 100时的相关代码分析

```kotlin
//CallServerInterceptor#intercept(chain)

exchange.noRequestBody()
if (!exchange.connection.isMultiplexed) {
    // If the "Expect: 100-continue" expectation wasn't met, prevent the HTTP/1 connection
    // from being reused. Otherwise we're still obligated to transmit the request body to
    // leave the connection in a consistent state.
    exchange.noNewExchangesOnConnection()
}
```

### 执行流程

当request(100-continue)没有收到code 100时会执行上述代码。

**isMultiplexed：**
* 使用http2时为true
* 使用http1.1时为false

**exchange.noNewExchangesOnConnection()理解**

调用以后会让当前使用的connection不能被链接池复用。对于此时connection上正在处理的请求没有影响。

**所以上述代码片段表示：**

如果当前使用http1.1，则不要复用这个connection.

### 原因

问了ai，但还是不太明白(**OKHTTP_TODO**)

* 为什么http1.1时不复用connection？
* 为什么http2时又能复用了?

## 跳过exchange.finishRequest()的情况
```kotlin
1.HttpMethod.permitsRequestBody(request.method) == false
2.requestBody != null
3.requestBody.isDuplex() == true
```
当1/2/3同时满足时，flushRequest()不会被调用，如何保证header被发送到了服务器(**OKHTTP_TODO**)？

okhttp默认的情况下1/2不会同时满足除非使用反射或者拦截器。
```kotlin
fun permitsRequestBody(method: String): Boolean = !(method == "GET" || method == "HEAD")
```
所以条件1变为：

method为GET或者HEAD。

当我们发起一个带body的GET请求时:
```kotlin
    val request = Request.Builder()
        .url("https://127.0.0.1:8443/hello2")
        .method("GET", requestBody)
        .build()
```
method()方法会抛出异常:
```kotlin
// Request.kt

open fun method(method: String, body: RequestBody?): Builder = apply {
    require(method.isNotEmpty()) {
        "method.isEmpty() == true"
    }
    if (body == null) {
        require(!HttpMethod.requiresRequestBody(method)) {
            "method $method must have a request body."
        }
    } else {
        // 这里检测：GET/HEAD请求不能含有body
        require(HttpMethod.permitsRequestBody(method)) {
            "method $method must not have a request body."
        }
    }
    this.method = method
    this.body = body
}
```

