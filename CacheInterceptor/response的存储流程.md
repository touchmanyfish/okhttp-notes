# response的存储流程

当如下两种场景时会执行response当存储流程:
* OkHttp手动构造的Validation返回了非304 response
* 没有reuse发生，OkHttp直接发起请求，收到了response

## 存储条件
rfc 7234定义
实际实现