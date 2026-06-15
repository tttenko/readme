```java
class JiraFeignErrorDecoder(
    private val objectMapper: ObjectMapper,
    private val messageProvider: MessageProvider
) : ErrorDecoder {

    private val log by logger()

    override fun decode(
        methodKey: String,
        response: Response
    ): Exception {

        val responseBody =
            response.body()
                ?.asInputStream()
                ?.bufferedReader(Charsets.UTF_8)
                ?.use { it.readText() }
                .orEmpty()

        log.error(
            "Jira request failed. methodKey: {}, status: {}",
            methodKey,
            response.status()
        )

        return when (response.status()) {

            TOO_MANY_REQUESTS.value() -> {
                RetryableException(
                    response.status(),
                    messageProvider["sbp.toomanyrequests.error"],
                    response.request().httpMethod(),
                    1000L,
                    response.request()
                )
            }

            BAD_REQUEST.value() -> {
                CreateIssueBadRequestException(
                    errorBody = runCatching {
                        objectMapper.readValue(
                            responseBody,
                            CreateIssueErrorDto::class.java
                        )
                    }.getOrElse {
                        CreateIssueErrorDto(
                            errorMessages = listOf(
                                responseBody.ifBlank {
                                    "Jira returned bad request without response body"
                                }
                            ),
                            errors = emptyMap()
                        )
                    },
                    rawBody = responseBody.ifBlank {
                        "Jira returned bad request without response body"
                    },
                )
            }

            else -> {
                JiraException(
                    status = SERVICE_UNAVAILABLE,
                    errorCode = response.status().toString(),
                    message = responseBody.ifBlank {
                        "Jira returned error without response body. Status: ${response.status()}"
                    },
                )
            }
        }
    }
}
```
