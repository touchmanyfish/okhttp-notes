# 使用RealConnection#connectTunnel()向代理服务器建立Tunnel


connectTunnel()方法先在建立rawSocket连接，然后在rawSocket上发送CONNECT请求建立通向代理服务器的通道，期间还会在同一个rawSocket上处理代理服务器可能发起的挑战认证。

**流程如下：**

1. 发起CONNECT请求
2. 处理n(n >= 0)个挑战
3. 服务器返回code 200，通道成功建立

## 处理挑战

服务器是否发起挑战取决于的配置。另外挑战是一个独立的机制，并不是和CONNECT请求绑定的。

### request

```kotlin
// RealConnection#createTunnel()

HTTP_PROXY_AUTH -> {
    nextRequest = route.address.proxyAuthenticator.authenticate(route, response)
        ?: throw IOException("Failed to authenticate with proxy")

    if ("close".equals(response.header("Connection"), ignoreCase = true)) {
        return nextRequest
    }
}
```

在开启tunnel的过程中，服务器发起了挑战，OkHttp将挑战信息(包含在response中)传递给authenticate()方法，库的使用者需要在其中解决挑战并将结果
放到新的request中，它会被再次发往服务器，完成一轮“挑战-解决”。服务器可能会发起0个1个或者多个挑战，这取决于服务器本次使用的 **挑战认证**。

挑战可以分为如下2类：

```text
// 被动
1. 客户端发起请求，不带任何解决挑战信息
2. 服务器发起挑战
3. 客户端被动的开始走挑战流程


// 主动
1. 客户端提前知道了服务器要使用的认证，以及这个认证要发起的挑战，并且这个认证支持主动解决挑战
2. 客户端发起请求时主动携带首次挑战的答案
3. 服务器收到答案以后两端开始后续挑战流程
```

显然第二种可以缩短挑战解决时间。所以在向代理服务器建立tunnel时，首次发起请求之执行如下代码:
```kotlin
 // RealConnection.kt

  @Throws(IOException::class)
  private fun createTunnelRequest(): Request {
    val proxyConnectRequest = Request.Builder()
        .url(route.address.url)
        .method("CONNECT", null)
        .header("Host", route.address.url.toHostHeader(includeDefaultPort = true))
        .header("Proxy-Connection", "Keep-Alive") // For HTTP/1.0 proxies like Squid.
        .header("User-Agent", userAgent)
        .build()

    val fakeAuthChallengeResponse = Response.Builder()
        .request(proxyConnectRequest)
        .protocol(Protocol.HTTP_1_1)
        .code(HTTP_PROXY_AUTH)
        .message("Preemptive Authenticate")
        .body(EMPTY_RESPONSE)
        .sentRequestAtMillis(-1L)
        .receivedResponseAtMillis(-1L)
        .header("Proxy-Authenticate", "OkHttp-Preemptive")
        .build()

    val authenticatedRequest = route.address.proxyAuthenticator
        .authenticate(route, fakeAuthChallengeResponse)

    return authenticatedRequest ?: proxyConnectRequest
  }
```

将fakeAuthChallengeResponse传递给proxyAuthenticator.authenticate(),让库的使用者有机会实现主动挑战。如果库的调用者知道服务器支持主动挑战，
则可以在这个方法中识别到"OkHttp-Preemptive"时，将挑战解决信息提前写入到request减少挑战流程时间。

### 挑战过程中代理服务器返回的“close”

挑战过程中服务器可能会因为一些原因返回“close”，此时OkHttp会：

1. 先让proxyAuthenticator.authenticate()计算出一个nextRequest
2. 然后断开rawSocket，并做相关清理
3. 使用nextRequest重走 **建立通道流程(连接socket，处理挑战)**

OkHttp最多会处理 MAX_TUNNEL_ATTEMPTS 次“close”，次数超过则 **建立通道失败**。

对于proxyAuthenticator.authenticate()的实现者，需要在其中识别"close",然后：

1. 如果提前知道服务器支持主动挑战则将挑战解决信息写入要返回的request
2. 如果不知道则正常构造request即可

### 深入理解MAX_TUNNEL_ATTEMPTS

在createTunnel()中只有服务器返回“close”才会导致返回一个非空request，而在外层的connectTunnel()识别到createTunnel()返回非空request时才
再次循环会循环，这说明:

1. MAX_TUNNEL_ATTEMPTS **只是处理"close"的最大次数**
2. 挑战过程中遇到的挑战题目多少MAX_TUNNEL_ATTEMPTS **管不了**！就是遇到一万个题目也没问题。

## 通道成功建立

当代理服务器在rawSocket上返回了code 200时，表示 **通道成功建立**。

