# OkHttp确定http协议版本与启动


## OkHttp确定http协议版本

```text
// 可以给OkHttpClient设置的协议组合

// 组合1
HTTP_1_1

// 组合2
HTTP_1_1/h2(HTTP_2)

// 组合3
h2c(H2_PRIOR_KNOWLEDGE)
```

### http

h2c和1.1是互相斥的，所以OkHttp:

1. 要么使用h2c
2. 要么使用1.1 
3. 绝不可能使用h2

### https

OkHttp使用ALPN选择协议(如果ALPN不能得出结论，则默认使用1.1)
```kotlin
// RealConnection#connectTls()

// Success! Save the handshake and the ALPN protocol.
val maybeProtocol = if (connectionSpec.supportsTlsExtensions) {
    Platform.get().getSelectedProtocol(sslSocket)
} else {
    null
}
```


## 启动确定版本的http协议

### HTTP/1.1

直接开始发送数据


### h2c

在rfc中有2种方式启动h2c:

1. 使用HTTP1.1 + Upgrade:h2c
2. 或者直接发送preface

OkHttp选择使用第2种。

```kotlin
  @Throws(IOException::class)
@JvmOverloads
fun start(sendConnectionPreface: Boolean = true, taskRunner: TaskRunner = TaskRunner.INSTANCE) {
    if (sendConnectionPreface) {
        writer.connectionPreface()
        writer.settings(okHttpSettings)
        val windowSize = okHttpSettings.initialWindowSize
        if (windowSize != DEFAULT_INITIAL_WINDOW_SIZE) {
            writer.windowUpdate(0, (windowSize - DEFAULT_INITIAL_WINDOW_SIZE).toLong())
        }
    }
    // Thread doesn't use client Dispatcher, since it is scoped potentially across clients via
    // ConnectionPool.
    taskRunner.newQueue().execute(name = connectionName, block = readerRunnable)
}
```

### h2

通过发送preface启动。

## TODO:tls(1.1 -> h2)

1. 不违反tls语法
2. 不违反upgrade语法
3. 对于这种tls上1.1升h2的升级路径没有被定义过