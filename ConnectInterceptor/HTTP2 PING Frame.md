# HTTP2 PING Frame

## rfc中定义的PING帧交互

* 发送端发送了PING(ACK=0)帧以后，接收端必须恢复一个PING(ACK=1)帧
* 每一端可以在没收到PING(ACK=0)帧的情况下主动发送PING(ACK=1)帧

## 机制的作用

rfc只定义了这个工具的样子，没有定义这个工具可以拿来干嘛。具体作用取决于HTTP2的实现者。

## okhttp中使用PING帧检测连接存活

```kotlin
// Http2Connection.kt

init {
    if (builder.pingIntervalMillis != 0) {
        val pingIntervalNanos = TimeUnit.MILLISECONDS.toNanos(builder.pingIntervalMillis.toLong())
        writerQueue.schedule("$connectionName ping", pingIntervalNanos) {
            val failDueToMissingPong = synchronized(this@Http2Connection) {
                // 根据rfc定义，当收到的PING帧数 >= 发送的PING帧时符合定义
                // 反推异常时的条件
                // received < sent
                if (intervalPongsReceived < intervalPingsSent) {
                    return@synchronized true
                } else {
                    intervalPingsSent++
                    return@synchronized false
                }
            }
            if (failDueToMissingPong) {
                // 连接gg了
                failConnection(null)
                return@schedule -1L
            } else {
                // 该发PING帧了！
                writePing(false, INTERVAL_PING, 0)
                return@schedule pingIntervalNanos
            }
        }
    }
}
``` 

OkHttp每隔pingIntervalNanos进行：

* 检测received/sent的PING帧数量是否有异常
* 有异常就使用failConnection()上报
* 没异常就发送PING帧