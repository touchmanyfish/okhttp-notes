# ConnectionSpecSelector

## connectionSpecs的配置/使用

1. 构造httpclient时配置
2. 发起call时，从传递给address
3. connect()方法从address中读取出来对sslSocket的当前配置进行约束

## “isCompatible()配置列表”

connectionSpecs中满足isCompatible()的spec构成一个isCompatible()配置列表。

## configureSecureSocket(sslSocket: SSLSocket)

configureSecureSocket()方法每次会使用列表中未被尝试过的配置来参与sslSocket的配置流程；
isFallbackPossible()用于判断是否还有可尝试的spec。


## connectionFailed(e: IOException)

**定义"可重试SSLException":**

```kotlin
// If the problem was a CertificateException from the X509TrustManager, do not retry.
e is SSLHandshakeException && e.cause is CertificateException -> false

// e.g. a certificate pinning error.
e is SSLPeerUnverifiedException -> false

// Retry for all other SSL failures.
e is SSLException -> true
```

**建立tls过程中的重试机制：**

当connectionTls()方法抛出异常满足如下条件时，OkHttp会尝试“isCompatible()配置列表”中的下一个配置来重新执行connectTls():

1. 还有可尝试的spec
2. e is "可重试SSLException"

所以这里的“重试”实际上是“当tls过程出错时，是用未尝试的可用配置重新进行tls过程”


