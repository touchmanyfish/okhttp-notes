# Content-Encoding

## 用法 
假设发送方做了如下配置：
```text
// 可以有一个或者多个取值
// 可能值:
// br
// gzip
// deflate
// identity 
Content-Encoding: deflate, gzip
```

**发送方：**
```text
1. 原始数据
2. 使用deflate压缩，生成产物d组
3. 将产物d组使用gzip压缩，生成产生g组
4. 将产生g组写入socket
```

数据流向:原始数据 -> deflate  -> gzip  -> socket。

数据会先使用Content-Encoding当值中靠前的压缩算法压缩。

**接收方:**
```text
5. 读取socket数据
6. 每当上一步中有足够数据使用gzip解压时，进行解压
7. 每当上一步中有足够数据使用deflate解压时，进行解压
8. 当socket中数据读取完毕，gzip和deflate解压完所有数据，整个过程结束
```

数据流向:socket -> gzip -> deflate -> 原始数据。

数据会先使用Content-Encoding当值中靠后的压缩算法解压。

### 如果同时配置了chunked
```text
Transfer-Encoding: chunked
Content-Encoding: deflate, gzip
```

**发送方：**

在最后一个压缩算法处理以后还需要分块，才能将数据写入socket。

**接收方：**

需要先将数据从分块中剥离出来，再交由最后一个压缩算法解压。

### 流式处理

总结一下Content-Encoding+Transfer-Encoding一起使用时的模型：
```text
// 发送方
原始数据 -> 一个或者多个算法压缩 -> chunked(如果开启了分块传输) -> socket

// 接收方
socket -> 从chunked中剥离数据(如果开启了分块传输) -> 一个或者多个算法解压 -> 原始数据
```

显然，数据总是“从一层到另一层”，**重点来了**：

某一层并不需要完整的处理完所有的上游数据，才将数据交给下一层；它可以将处理了部分上层数据的结果交给下一层。

## Accept-Encoding

```text
// 例子
Accept-Encoding: gzip, deflate, br
```

“Accept-Encoding”用于向接收方表明自己能处理的压缩算法。发送方是否使用这些压缩算法处理发送内容是可选的。

