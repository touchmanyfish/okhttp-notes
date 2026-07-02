# http协议的升级相关操作

## http中版本兼容

在相同major下：

1. minor低的收到minor高的报文，只处理其中自己认识的部分
2. minor高的收到minor低的部分，需要能完全处理

## 服务器收到请求以后可以执行的rfc操作

### 理解

在rfc定义中，服务器收到请求以后，当前状态下可执行的操作可能不止一种，服务器可以根据自己的支持程度和喜好选择操作执行。

### 101 协议升级

#### 作用

用协商的方式将 **指定版本的http协议** 升级为 **另一个协议**。当服务器收到客户端的Upgrade请求时，如果它自己满足条件则可以选择执行101升级协议。

#### 升级流程

```text
// 指定http版本
a.b

// 其他协议
C/D/E

// 已有定义的升级路线
path1 : a.b -> C/1.0
path2 : a.b -> D/1.1
path3 : a.b -> E/2.1

// 客户端(a.b)
// 支持的升级路线
path1/path2

// 服务器(a.b)
// 服务器支持的升级路线
path1/path2/path3
```
客户端选择期望且自己支持的升级路径通过Upgrade参数告知服务器:

```text
GET /chat HTTP/a.b
Host: api.example.com
Connection: Upgrade
Upgrade: C/1.0, D/1.1
```
服务器根据喜好选择自己支持的升级路线 **"path1 : a.b -> C/1.0"** ，并告Upgrade参数知客户端:
```text
HTTP/a.b 101 Switching Protocols
Upgrade: C/1.0
Connection: Upgrade
```
后续双方按照升级路径对应的定义进行操作。

#### 不执行101升级协议

101协议的执行是可选的，服务器在收到客户端的Upgrade请求时也可以选择执行其他rfc允许的操作。

#### Upgrade参数

客户端在其中设置自己期望升级到的目标协议。服务器在其中指定从客户端期望的选定的目标协议。
```text
// 客户端
Upgrade:期望升级到的协议列表(plist)

// 服务器
Upgrade:从plist中选择的一个协议
```

### 返回code 200

服务器在读取到客户端当前请求的http版本号以后可以选择使用相同的http版本进行回复。

### 返回code 426

```text
// 服务器返回的426报文例子

HTTP/1.1 426 Required Update
Connection: Upgrade
Upgrade: HTTP/1.1, HTTP/1.2, h2c, Websocket
Content-Length: 0
```

当收到客户端的请求时，，服务器 **可以** 通过返回426来要求切换到一个新协议上，新协议需要：

1. 服务器自己支持这个协议
2. 如果客户端当前协议无法满足业务，必须切换到支持业务的版本
3. 迁移路径存在定义

