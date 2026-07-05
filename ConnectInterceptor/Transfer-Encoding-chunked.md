# Transfer-Encoding: chunked

在HTTP1.0中，如果要支持keep-alive长连接，服务器需要向客户端返回content-length。由于很难提前计算动态内容的大小，所以服务器需要先完整的生成动态内容才能拿到需要的content-length。但是当时的内存资源紧张，有时并不支持这种操作。于是服务器不得不选择禁用对keep-alive的支持。

为了解决这个问题诞生了Transfer-Encoding。

```text
// 表示内容对内容进行分块传输
// 虽然“Transfer-Encoding”还能有其他取值，但是常用的就是上述取值。
Transfer-Encoding: chunked

```

发送方使用上述header，表示自己会对内容进行分块传输。此时内容结构如下：

```text
HTTP/1.1 200 OK
Content-Type: text/plain
Transfer-Encoding: chunked
Trailer: X-Response-Checksum, X-Stream-Error-Code
Date: Sat, 04 Jul 2026 09:45:00 GMT

[十六进制长度]\r\n
[二进制数据]\r\n

[十六进制长度]\r\n
[二进制数据]\r\n

...

0\r\n
[trailer1]\r\n
[trailer2]\r\n
...
\r\n

```

**千万不要将"Transfer-Encoding: chunked"理解为**：

1. 发送方发送这个header期望接收方使用分块传输
2. 接收方收到以后通过了这个请求，并使用分块传输

上述理解是错误的，实际上只要你使用了这个header，就表明你即将使用分块传输。

## chunked块

每个chunked块包含:

* 块的长度（**必须为十六进制字符串**）
* 对应的二进制数据

当接受方识别到“0\r\n”时表示所有chunked块已接受完毕。

## trailer

对于使用HTTP的双方，其中一方可以使用如下header，表明自己期望另一端在使用分块传输尾部携带trailers(神奇的是这里并不能指定期望的trailers名字):

```text
TE: trailers

```

另一方 **可以** 忽略这个header，也 **可以** 选择响应这个要求，此时 **必须** 同时返回:

* Trailer header:声明在分块传输尾部会附带哪些trailer（注意字段名为单数 **`Trailer`**）
* 分块传输尾部附带在header中声明的trailer

```text
Trailer: trailer1, trailer2..

0\r\n
[trailer1]\r\n
[trailer2]\r\n
...
\r\n 

```

### Trailer 的限制场景（协议红线）

并不是所有的 HTTP Header 都可以放入分块尾部的 Trailer 中。根据 RFC 规范，以下几类涉及消息框架、路由、安全或认证的 Header **绝对不允许（MUST NOT）** 作为 Trailer 发送。如果包含，接收方必须将其忽略或视为协议错误：

* **消息框架相关**：`Transfer-Encoding`、`Content-Length`、`Trailer`
* **路由与连接相关**：`Connection`、`Host`
* **安全与隐私相关**：`Authorization`、`Set-Cookie`（防止走私缓存污染与请求走私攻击）

## chunked和“Content-Length”同时出现时

对于这种情况，在rfc中有如下规矩：

1. 接收方必须忽略“Content-Length”
2. 如果接收方是proxy，需要删除“Content-Length”
3. 接收方可以选择返回 400 Bad Request 并关闭连接

## 真实的例子

假设客户端发了 TE: trailers

```text
HTTP/1.1 200 OK\r\n
Content-Type: text/plain\r\n
Transfer-Encoding: chunked\r\n
Trailer: X-Log-MD5, X-Server-Status\r\n
Date: Sat, 04 Jul 2026 10:45:00 GMT\r\n
\r\n
12\r\n                                      <-- 十六进制的12，代表十进制的18字节
Chunk number one. \r\n
11\r\n                                      <-- 十六进制的11，代表十进制的17字节
Chunk number two.\r\n
0\r\n
X-Log-MD5: 9a45b6c7d8e9f012\r\n
X-Server-Status: SUCCESS\r\n
\r\n

```