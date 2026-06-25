# CacheStrategy#isCacheable()

这个方法用于判断收到的response是否具备被缓存的 **[部分条件]**：
* **true**:reponse具备了被缓存的部分条件；是否能被缓存还需要进一步判断。
* **false**:response不会被缓存

## 内部进行的判断

### 1.判断code是否是cacheable的

cacheable的code分为2类

**默认cacheable的code（ [rfc 7231 Overview of Status Codes](https://www.rfc-editor.org/rfc/rfc7231.html#page-48)）**:
```text
200, 203, 204, 300, 301, 404, 405, 410, 414, 501
```

**注意！虽然206默认也是cacheable(rfc 7231)，但是OkHttp选择了不处理它！**

**满足一定条件就能cacheable的code:**
```text
302, 307
```
当302/307满足如下条件时，就具有cacheable([rfc 2616](https://www.rfc-editor.org/rfc/rfc2616.html#section-10.3.8)):
```text
This response is only cacheable if indicated by a Cache-Control or Expires header
   field
```

在OkHttp中对应的处理代码:
```kotlin
//CacheStrategy.isCacheable()

// 302
HTTP_MOVED_TEMP,
// 307
StatusLine.HTTP_TEMP_REDIRECT -> {
    // These codes can only be cached with the right response headers.
    // http://tools.ietf.org/html/rfc7234#section-3
    // s-maxage is not checked because OkHttp is a private cache that should ignore s-maxage.
    
    
    if (
        //indicated by 对应定义中的Expires
        response.header("Expires") == null &&
        
        // 对应定义中的indicated by Cache-Control
        response.cacheControl.maxAgeSeconds == -1 &&
        !response.cacheControl.isPublic &&
        !response.cacheControl.isPrivate) {
        return false
    }
}
```

在rfc中302对应的含义有如下变化:

rfc 1945:302 Moved Temporarily

rfc 2616:302 Found

rfc 9110:302 Found

OkHttp选择了rfc 1945中使用的含义来给302取常量名：HTTP_MOVED_TEMP。

### 2.判断是否没有no-store

```kotlin
// A 'no-store' directive on request or response prevents the response from being cached.
return !response.cacheControl.noStore && !request.cacheControl.noStore
```