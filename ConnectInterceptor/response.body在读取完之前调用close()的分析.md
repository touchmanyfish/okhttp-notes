# response.body在读取完之前调用close()的分析

分析为了让connection可以被复用，body.close()中做的处理

## http1.1

```text
// HTTP/1.1 connection

1.发送请求
2.收到响应后立即调用response.body.close()
```

调用close()以后，rawSocket.inputStream中可能有response数据。我们需要将本次响应的数据从rawSocket.inputStream全部读取出来，如果这个
connection被复用，就可以从其中正确的读取下一个请求的响应。

在读取时优先使用 **协议层面的边界** 来判断数据是否读取结束;如果检测到对方主动发送EOF，需要手动将connection标记为不可复用。

### response.body

```text
// response.body有如下3种情况
// 调用read()或者close()时，实际会调用到这3种Source

body -> RealResponseBody -> FixedLengthSource
                            ChunkedSource
                            newUnknownLengthSource
```

### source.read()

FixedLengthSource与ChunkedSource的read()的实现逻辑如下：

1. 优先使用非EOF机制判断是否读取结束
2. 判断到对方主动发送了EOF，才将connection标记为不可复用(noNewExchanges == true)

```kotlin
// 举例
// 假设响应中包含“Content-Length”
// FixedLengthSource#read()

override fun read(sink: Buffer, byteCount: Long): Long {
    require(byteCount >= 0L) { "byteCount < 0: $byteCount" }
    check(!closed) { "closed" }
    
    // 客户端提前知道了响应body的长度
    // bytesRemaining表示body还未读取的量
    // 为0时表示响应的body读取完了
    if (bytesRemaining == 0L) return -1

    val read = super.read(sink, minOf(bytesRemaining, byteCount))
    if (read == -1L) {
        // 响应可能被读取完，但是对面发了EOF，这意味着rawSocket的输入流被关闭了
        // 这种HTTP/1.1 connection必然没法被复用的
        // 这里标记一下conncection
        connection.noNewExchanges() // The server didn't supply the promised content length.
        val e = ProtocolException("unexpected end of stream")
        responseBodyComplete()
        throw e
    }

    // 判断同上
    bytesRemaining -= read
    if (bytesRemaining == 0L) {
        responseBodyComplete()
    }
    return read
}

```

### source.close()

FixedLengthSource与ChunkedSource的close()的实现逻辑如下：

1. 如果响应中还有未读取的数据，在规定时间将它们全部读取出来
2. 如果规定时间内未完成或者读取时发生了异常，则将connection标记为不可复用(noNewExchanges == true)

将剩余数据全部读取完毕是为了connection被复用以后，不会错误的读取到上一个请求的数据；

而读取加上时间限制是为了避免遇到大响应时剩余数据读取过长，超过了
新创建connection的时间，这种情况只好放弃这个connection。你也可以理解为：承载大响应的connection更不容易被复用。

### UnknownLengthSource()的特殊处理

如果服务器没有使用分块传输并且也没返回"Content-Length",OkHttp会使用UnknownLengthSource(),并且会将connection标记为不可复用(noNewExchanges == true)。

## http2.0

```text
// HTTP/2.0 connection

1.发送请求
2.收到响应后立即调用response.body.close()
```

### 发送窗口

HTTP/2.0的两端A和B，A指定一个窗口来限制B。这个窗口的大小实质就是“A未向B通报的已收到数据量”。B在向A发送数据的过程中，如果A未通报的数据量累计到窗口大小，
B需要停止发送数据；A可以通过向B通报已收到数据来降低这个未通报量。

### close()分析

主要干了一件事：

以一种不会影响rawSocket.inputStream的方式把当前请求在HTTP/2.0中对应的stream(并不是rawSocket中的inputStream)掐了。这个掐不会导致当前连接上
其他请求被关闭，也不会影响当前connection的复用。

```kotlin
//FramingSource.kt

override fun close() {
    val bytesDiscarded: Long

    synchronized(this@Http2Stream) {
        // 标记一下，readBuffer不会再有数据写入
        closed = true
        // 获取从服务器收到的但是还未被 reader 读取的数据
        bytesDiscarded = readBuffer.size
        // 抛弃这些数据
        readBuffer.clear()

        this@Http2Stream.notifyAll() // TODO(jwilson): Unnecessary?
    }

    if (bytesDiscarded > 0L) {
        // 显然 bytesDiscarded 是未通报的
        // 如果当前未通报量超过阈值则通报给服务器
        updateConnectionFlowControl(bytesDiscarded)
    }

    // 它会通过向服务器发送 RST_STREAM 帧 来把当前请求对应的 stream 掐断。这种方式不会影响底层的connection，也不影响其他并发的请求流。
    cancelStreamIfNecessary()
}
```
