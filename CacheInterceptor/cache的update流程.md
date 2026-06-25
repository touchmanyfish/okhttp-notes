# Cache#update()

OkHttp收到GET request以后，会先尝试在本地查找满足reuse条件的stored response，但是只找到满足其中部分条件的:
* uri match
* method match
* varys match

于是根据这个request中携带的validator，手动帮这个request构造一个条件请求发往下一跳。当收到304以后，Okhttp会调用
Cache#update()来更新本地缓存。

## Okhttp实现 vs rfc 7234

rfc 7234在 [4.3.4.  Freshening Stored Responses upon Validation](https://www.rfc-editor.org/rfc/rfc7234#section-4.3.4)中
定义了当收到304时如何更新缓存。Okhttp的实现略有不同。

### 更新范围

rfc 7234中会根据304 response中的validator来select一个或者多个待更新的stored reponse。OkHttp则总是使用在发起validation以前就提前选好的
一个cache response。

### Warning header fields处理
```text
 If a stored response is selected for update, the cache MUST:

   o  delete any Warning header fields in the stored response with
      warn-code 1xx (see Section 5.5);

   o  retain any Warning header fields in the stored response with
      warn-code 2xx; and,

   o  use other header fields provided in the 304 (Not Modified)
      response to replace all instances of the corresponding header
      fields in the stored response.
```
这是rfc 7234定义的header方式；OkHttp则使用直接覆盖来更新:
```kotlin
// Cache#update()
internal fun update(cached: Response, network: Response) {
    val entry = Entry(network)
    val snapshot = (cached.body as CacheResponseBody).snapshot
    var editor: DiskLruCache.Editor? = null
    try {
        editor = snapshot.edit() ?: return // edit() returns null if snapshot is not current.
        // 304 response直接覆盖掉cache response的header
        entry.writeTo(editor)
        editor.commit()
    } catch (_: IOException) {
        abortQuietly(editor)
    }
}

//Entry#writeTo()
@Throws(IOException::class)
fun writeTo(editor: DiskLruCache.Editor) {
    // 指定覆盖ENTRY_METADATA部分
    editor.newSink(ENTRY_METADATA).buffer().use { sink ->
        // 这里时覆盖内容
        sink.writeUtf8(url.toString()).writeByte('\n'.toInt())
        sink.writeUtf8(requestMethod).writeByte('\n'.toInt())
        sink.writeDecimalLong(varyHeaders.size.toLong()).writeByte('\n'.toInt())
        for (i in 0 until varyHeaders.size) {
            sink.writeUtf8(varyHeaders.name(i))
                .writeUtf8(": ")
                .writeUtf8(varyHeaders.value(i))
                .writeByte('\n'.toInt())
//.. 
```