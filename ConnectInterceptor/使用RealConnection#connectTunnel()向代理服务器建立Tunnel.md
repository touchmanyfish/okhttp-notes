# ConnectionSpec

将ConnectionSpec中的配置与sslSocket原本的配置求一个 **交集**，如果这个ConnectionSpec能通过isCompatible()验证，则可以将交集中的
**部分配置** 通过apply()方法设置给sslSocket.

## ConnectionSpec与sslSocket中已有配置的交集

**[ConnectionSpec中的配置]**

* *isTls* — 客户端是否启用 TLS (Boolean)
* *supportsTlsExtensions* — 是否支持 TLS 扩展 (Boolean)
* *tlsVersionsAsString* — 客户端支持的 TLS 版本集合
* *cipherSuitesAsString* — 客户端支持的加密套件集合

**[sslSocket中已有配置]**

* *enabledProtocols* — 约束允许的 TLS 版本集合
* *enabledCipherSuites* — 约束允许的加密套件集合

**[预定义中间变量]**


$$I = enabledProtocols \cap tlsVersionsAsString$$

$$\text{signalCond} = (isFallback = \text{true}) \land (\text{TLS\_FALLBACK\_SCSV} \in supportedCipherSuites)$$

**[配置的交集]**


$$\text{result\_isTls} = isTls$$

$$\text{result\_tlsVersions} = I$$

$$\text{result\_cipherSuites} = \begin{cases} (enabledCipherSuites \cap cipherSuitesAsString) \cup \{\text{TLS\_FALLBACK\_SCSV}\}, & \text{当 } \text{signalCond} \text{ 成立} \\ enabledCipherSuites \cap cipherSuitesAsString, & \text{其他情况} \end{cases}$$

## isCompatible(socket: SSLSocket)

当满足如下条件时isCompatible()返回true:

$$(tlsVersionsAsString \neq \emptyset \rightarrow tlsVersionsAsString \cap enabledProtocols \neq \emptyset) \land (cipherSuitesAsString \neq \emptyset \rightarrow cipherSuitesAsString \cap enabledCipherSuites \neq \emptyset)$$

## 应用ConnectionSpec中的配置

ConnectionSpec如果能通过isCompatible()的检查，则可以调用apply()方法将 交集中的配置 应用到sslSocket中。

### 设置tlsVersion/cipherSuites

对socket应用如下设置：

$$socket.enabledProtocols = \text{result\_tlsVersions}$$

$$socket.enabledCipherSuites = \text{result\_cipherSuites}$$

### 设置tlsExtensions

connectionSpec.apply()方法中并不对tlsExtensions进行设置。OkHttp在apply()方法外部通过如下代码设置:

```kotlin
if (connectionSpec.supportsTlsExtensions) {
    Platform.get().configureTlsExtensions(sslSocket, address.url.host, address.protocols)
}

```

## sslSocket中的配置注意点

```text
// sslSocket中指定的协议
sslSocket.enabledProtocols

// sslSocket中指定的密码套件
sslSocket.enabledCipherSuites

// sslSocket支持的密码套件
sslSocket.supportedCipherSuites

```

enabledCipherSuites是supportedCipherSuites的子集，尝试设置一个不支持的套件到enabledCipherSuites中会报错，所以apply时不用求
result_cipherSuites与supportedCipherSuites的交集