```java

if (request.jql.contains("\"Epic Link\"", ignoreCase = true)) {
    monitoringTaskSearchRequests.incrementAndGet()

    if (monitoringTaskSearchShouldFail) {
        sendMonitoringTaskSearchError(exchange)
        return
    }
}
```
