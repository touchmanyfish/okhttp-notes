## cacheResponseAge()

```kotlin
/**
 * Returns the current age of the response, in milliseconds. The calculation is specified by RFC
 * 7234, 4.2.3 Calculating Age.
 */
private fun cacheResponseAge(): Long {
    val servedDate = this.servedDate
    val apparentReceivedAge = if (servedDate != null) {
        maxOf(0, receivedResponseMillis - servedDate.time)
    } else {
        0
    }

    val receivedAge = if (ageSeconds != -1) {
        maxOf(apparentReceivedAge, SECONDS.toMillis(ageSeconds.toLong()))
    } else {
        apparentReceivedAge
    }

    val responseDuration = receivedResponseMillis - sentRequestMillis
    val residentDuration = nowMillis - receivedResponseMillis
    return receivedAge + responseDuration + residentDuration
}
```
上述计算规则实际是按照[RFC 2616 13.2.3 Age Calculations](https://datatracker.ietf.org/doc/html/rfc2616#section-13.2.3)实现的，
而且查看[history:ResponseStrategy.java](https://github.com/square/okhttp/blob/350c43b6fe02401a73f967d9ef322061638b372a/okhttp/src/main/java/com/squareup/okhttp/internal/http/ResponseStrategy.java)
发现，从最开始就是这个写法。
```text
apparent_age = max(0, response_time - date_value);
corrected_received_age = max(apparent_age, age_value);
response_delay = response_time - request_time;
corrected_initial_age = corrected_received_age + response_delay;
resident_time = now - response_time;
current_age   = corrected_initial_age + resident_time;

// 变形1
current_age   = corrected_received_age + response_delay + resident_time;

// 最终变形
current_age   = max(apparent_age, age_value) + response_delay + resident_time;
```
并不是按照[rfc 7234  4.2.3 Calculating Age](https://www.rfc-editor.org/rfc/rfc7234#section-4.2.3)来写的，但是研究发现，上述
```text
// 原始公式
current_age = corrected_initial_age + resident_time;

// 变形
current_age = max(apparent_age, corrected_age_value) + resident_time;

// 最终变形
current_age = max(apparent_age, age_value + response_delay) + resident_time;
```

## 问题

为啥会这样呢？不知道哎(**OKHTTP_TODO**)