# Socket#isHealthy()方法分析

## 判定分析

OkHttp会先以“socket的输入流上的业务数据”为前提，然后在1ms内读取socket的输入流缓冲区,此时:

1. 立即读到了EOF,表明对端已关闭连接。
2. 直接抛出异常，表明连接发生故障。
3. 读到了业务数据，这与期望的不符,显然这个连接有问题
4. 啥也没读到，暂时认为连接是安全的

所以OkHttp实际上是在利用socket的输入缓冲区去感知连接是否有问题。所以当Socket#isHealthy()返回true时表示OkHttp当前没发现这个连接有啥问题，并不代表
这个连接真的没问题。

## Okhttp只在非GET方法时使用这个方法

```kotlin
fun find(
    client: OkHttpClient,
    chain: RealInterceptorChain
  ): ExchangeCodec {
    try {
      val resultConnection = findHealthyConnection(
          connectTimeout = chain.connectTimeoutMillis,
          readTimeout = chain.readTimeoutMillis,
          writeTimeout = chain.writeTimeoutMillis,
          pingIntervalMillis = client.pingIntervalMillis,
          connectionRetryEnabled = client.retryOnConnectionFailure,
          // 这里
          doExtensiveHealthChecks = chain.request.method != "GET"
      )
      return resultConnection.newCodec(client, chain)
    } catch (e: RouteException) {
      trackFailure(e.lastConnectException)
      throw e
    } catch (e: IOException) {
      trackFailure(e)
      throw RouteException(e)
    }
  }
```

### 原因分析(TODO)

OkHttp维护者在stackoverflow的 [回答](https://stackoverflow.com/questions/40933697/okhttp-post-slower-than-get-and-why-health-check-when-not-get-method?utm_source=chatgpt.com):

```text
Just a guess but GET is a safe Method, so it doesn't hurt to try, fail and then retry.

https://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html

For POST it makes sense to check the connection is healthy before attempting the request.
```

### 为什么非GET的幂等方法也要执行ExtensiveHealthChecks？

TODO
