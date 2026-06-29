# HTTP Challenge-Response Authentication Architecture

## 简介

当向服务器发起请求的时候，服务器可以返回 **挑战(Challenge)**,客户端需要在下一次的request中携带解决挑战的信息。这种“挑战-解决”会经历
一轮或者多轮，这取决于服务器使用的 **认证类型**。最终服务器返回code 200表示挑战通过。

## 认证挑战模型

**首轮：**

* 客户端request(其中可以携带预先知道的挑战的解决办法)到达服务器
* 服务器在response中返回挑战
* 客户端发送解决挑战的request

**n(n>=0)轮：**

* 客户端request到达服务器
* 服务器在response中返回挑战
* 客户端发送解决挑战的request

**最终：**

在某次客户端发送解决挑战的request以后，服务器返回了code 200

## 单个挑战题目的认证

对于 **认证挑战模型** 中 n == 0 时的情况，例如：

* Basic 认证
* Digest 摘要认证

```text
// 举例：发起CONNECT请求遇到Basic认证流程

[ 首轮 ]：
 1. 客户端：空手发送普通的 CONNECT 请求
 2. 服务器：拦截，返回挑战（407 Proxy Authentication Required）
           Header 携带：Proxy-Authenticate: Basic realm="xxx"
 3. 客户端：触发 Authenticator，立刻计算解决挑战的 Request

   │
   ▼
[ 最终 ]：
 客户端：发送带有“挑战答案（Basic 认证头）”的新 Request
 服务器：验算通过，返回 200 OK
```

## 401/407

401/407共用同一个挑战模型；下面是它们的区别。

### 401

表示源服务器发起了挑战。

**服务器发送挑战：**

* code 401
* 在header "WWW-Authenticate"中携带挑战信息

**客户端解决挑战：**

* 在header "Authorization"中携带解决信息


```text
// 场景举例

1. 客户端发送 GET /index.html。
2. 源服务器返回 401，带上 WWW-Authenticate: Basic realm="Git"。
3. 客户端在下一轮解决挑战，必须带上：Authorization: Basic dXNlcjp...
```

### 407

表示代理服务器发起了挑战。

**服务器发送挑战：**

* code 407
* 在header "Proxy-Authenticate"中携带挑战信息

**客户端解决挑战：**

* 在header "Proxy-Authorization"中携带解决信息

注意！服务器使用的header是 **cate** 结尾，客户端使用的header是 **tion** 结尾


```text
// 场景举例

1. 客户端发送 CONNECT github.com:443。
2. 代理服务器拦截返回 407，带上 Proxy-Authenticate: Basic realm="CompanyProxy"。
3. 客户端在下一轮解决挑战，必须带上：Proxy-Authorization: Basic dXNlcjp...。
```

## 多个挑战题目的认证
对于 **认证挑战模型** 中 n > 0 时的情况,例如：

* NTLM认证

```text
// 举例:发起CONNECT请求遇到NTLM认证流程

[ 首轮 ]：（客户端试探，服务器出第 1 道题）
 1. 客户端：发送普通的 CONNECT 请求（空手去）。
 2. 服务器：拦截，返回 407，并告诉客户端：“我支持 NTLM 认证”。
 3. 客户端：解出第 1 个挑战，构建【Type 1 协商消息】（声明自己的安全特性）发过去。

   │
   ▼
[ n 轮 (n = 1) ]：（服务器出第 2 道题，客户端解最核心的题）
 1. 服务器：收到 Type 1，丢回第 2 个 407 挑战。
           Header 携带：【Type 2 质询消息】（里面包含服务器临时生成的动态随机数 Challenge）。
 2. 客户端：触发 Authenticator，动用密码哈希去加密这个随机数，
           构建出极其复杂的【Type 3 认证消息】发送解决挑战的 Request。

   │
   ▼
[ 最终 ]：
 服务器：收到 Type 3，在后台对撞成功，确信客户端知道密码，满意地返回 200 OK。
```

## 自适应挑战的认证

根据情况动态决定上述模型中n的具体数值。

一种可能的使用场景是：

* 如果检测到你发起请求的环境安全，则需要解决的挑战数目就小
* 如果不安全，需要解决的挑战数目就多

## 抢占式/非抢占式

### 非抢占式

首次发起某个请求时不携带任何挑战解决信息，等待服务器返回挑战信息。

### 抢占式

提前知道了服务器会进行的挑战，在首次发起请求时就携带挑战解决信息。不过这需要认证方式支持才行。

