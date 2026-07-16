# HTTP2 GOAWAY Frame

## rfc中的定义

### 作用

HTTP2的Endpoint发送GOAWAY帧来宣告连接即将关闭，通知对端停止创建新流，但允许已创建的存量流继续传输完毕。

### 帧结构中的关键字段

**Last-Stream-ID：**

发送端保证 **已经处理、正在处理、或可能处理** 的对端发起的流的id最大值；这个值不影响自己发起的流。

**Error Code：**

错误码，告诉对方为什么要关闭连接。

**Additional Debug Data:**

可选字段，调试信息

### 交互流程

```text
// HTTP2
A <--------> B

// A发起的流的stream id
3 5 7 9 11

// B发起的流的stream id
2 4 6 8 10
```

接着B向A发送GOAWAY(Last-Stream-Id:9),表示

1. B保证 **已经处理、正在处理、或可能处理** A发起的流中id <= Last-Stream-Id:9的那些([3 5 7 9])
2. B不会处理A_stream[11]
3. B自己发起的[2 4 6 8 10]不受任何影响

对于A_stream[id > Last-Stream-Id:9],因为它们没有被B处理过，所以A可以根据自己的逻辑选择为其发起重试。

### 注意点

HTTP2的任意一端(EndPoint)都能发GOAWAY帧，不要错误的认为只有服务器能发。

## OkHttp中的相关处理

### 发送GOAWAY帧

```kotlin
@Throws(IOException::class)
fun shutdown(statusCode: ErrorCode) {
    synchronized(writer) {
        val lastGoodStreamId: Int
        synchronized(this) {
            if (isShutdown) {
                return
            }
            isShutdown = true
            lastGoodStreamId = this.lastGoodStreamId
        }
        // TODO: propagate exception message into debugData.
        // TODO: configure a timeout on the reader so that it doesn’t block forever.
        writer.goAway(lastGoodStreamId, statusCode, EMPTY_BYTE_ARRAY)
    }
}
```

### 接收GOAWAY帧

```kotlin
override fun goAway(
    lastGoodStreamId: Int,
    errorCode: ErrorCode,
    debugData: ByteString
) {
    // OkHttp在这个方法里根本不鸟errorCode和debugData
    
    if (debugData.size > 0) {
        // TODO: log the debugData
    }

    // Copy the streams first. We don't want to hold a lock when we call receiveRstStream().
    val streamsCopy: Array<Http2Stream>
    synchronized(this@Http2Connection) {
        streamsCopy = streams.values.toTypedArray()
        // 服务器发起流的场景只有Server Push，但是OkHttp并没有实现这个功能。
        // 于是直接将当前connection标记为:不可产生新的stream+不可复用
        isShutdown = true
    }

    // Fail all streams created after the last good stream ID.
    for (http2Stream in streamsCopy) {
        if (http2Stream.id > lastGoodStreamId && http2Stream.isLocallyInitiated) {
            // 把自己发起的id > Last-Stream-Id的流都给掐了
            // 其他流让它们继续跑着
            http2Stream.receiveRstStream(REFUSED_STREAM)
            removeStream(http2Stream.id)
        }
    }
}
```


