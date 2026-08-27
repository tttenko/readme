```java

private fun sendMonitoringTaskSearchError(exchange: HttpExchange) {
    val response = """
        {
          "message": "Integration test monitoring Task search failure"
        }
    """.trimIndent().toByteArray(StandardCharsets.UTF_8)

    exchange.responseHeaders.set("Content-Type", "application/json")
    exchange.sendResponseHeaders(500, response.size.toLong())

    exchange.responseBody.use { output ->
        output.write(response)
    }
}
```
