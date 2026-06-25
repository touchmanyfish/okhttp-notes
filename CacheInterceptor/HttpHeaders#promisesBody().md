# HttpHeaders#promisesBody()

这个方法用于判定response是否**可能**有body；**注意，这里是可能而不是一定！**

## 判断流程

1. HEAD
2. code
3. Content-Length/Transfer-Encoding

### 1.通过HEAD判断
```kotlin
// HEAD requests never yield a body regardless of the response headers.
if (request.method == "HEAD") { 
    return false
}
```
根据HEAD的定义([rfc 7231 4.3.2.  HEAD](https://www.rfc-editor.org/rfc/rfc7231.html#section-4.3.2)),对应的response没有body.

### 2.通过code判断

```kotlin
val responseCode = code
if ((responseCode < HTTP_CONTINUE || responseCode >= 200) &&
    responseCode != HTTP_NO_CONTENT &&
    responseCode != HTTP_NOT_MODIFIED) { 
    return true
}
```
先来看一下状态码范围

| 状态码范围         | 范围名称 (Category)           | 是否有 Body          | 说明                                  |
|:--------------|:--------------------------|:------------------|:------------------------------------|
| **< 100**     | **非标准/扩展 (Non-standard)** | **可能有**           | 协议未定义，OkHttp 为兼容性考虑通常尝试读取。          |
| **100 - 199** | **信息性响应 (Informational)** | **无**             | 如 100 Continue。仅作为中间过渡，不包含主体。       |
| **200 - 299** | **成功响应 (Successful)**     | **有 (除 204/205)** | 表示请求成功。204 (No Content) 明确规定无 Body。 |
| **300 - 399** | **重定向 (Redirection)**     | **可能有(除 304)**    | 通常包含重定向目标的超链接信息。304 明确规定无 Body。     |
| **400 - 499** | **客户端错误 (Client Error)**  | **有**             | 如 404。通常包含错误原因的解释页面或 JSON。          |
| **500 - 599** | **服务器错误 (Server Error)**  | **有**             | 如 500。通常包含服务器故障的描述信息。               |

OkHttp在上述表格中被定义为“有”/“可能有”的情况返回true。

另外当205时也返回true，因为：
```kotlin
// gemini

在某些 RFC 修订版本中，205 也被要求不能携带 Body。但 OkHttp 的代码逻辑更偏向于“实战”——由于 205 在现代 Web 开发中极罕见，且绝大多数 2xx 
响应都应该有 Body（或至少允许有），所以它只对最明确的 204 进行了硬编码排除。
```

### 3.通过Content-Length/Transfer-Encoding处理兼容的情况
```kotlin
// If the Content-Length or Transfer-Encoding headers disagree with the response code, the
// response is malformed. For best compatibility, we honor the headers.
if (headersContentLength() != -1L ||
    "chunked".equals(header("Transfer-Encoding"), ignoreCase = true)) { 
    return true
}
```
如果通过code判断没有body，但是通过Content-Length/Transfer-Encoding判断有，此时以header为准。

**逻辑背景(gemini)**

根据 HTTP 协议（如 RFC 7230），像 204 No Content 或 304 Not Modified 这种状态码是不允许有 Body 的。但在现实世界中，由于各种后端框架或代理服务器（Proxy）的配置错误，偶尔会出现“虽然状态码说没内容，但 Header 里却给出了长度”的情况。

**为什么要这么做？(gemini)**

如果服务器发出了 Content-Length: 100 但 OkHttp 因为状态码是 204 而忽略它，那么这 100 字节的数据就会残留在 Socket 缓冲区中。当下一个请求复用这个连接时，它会读到上一个请求留下的“脏数据”，导致整个网络连接发生不可预知的崩溃。

举例:[java.net.ProtocolException: unexpected end of stream with 204 response #1490](https://github.com/square/okhttp/issues/1490)