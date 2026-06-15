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

        val rawBody =
            readResponseBody(response)

        log.error(
            "Jira request failed. methodKey: {}, status: {}, body: {}",
            methodKey,
            response.status(),
            rawBody
        )

        return when (response.status()) {

            TOO_MANY_REQUESTS.value() -> {

                RetryableException(
                    status = response.status(),
                    message = messageProvider["sbp.toomanyrequests.error"],
                    httpMethod = response.request().httpMethod(),
                    retryAfter = 1000L,
                    request = response.request()
                )
            }

            BAD_REQUEST.value() -> {

                if (isCreateIssueMethod(methodKey)) {

                    CreateIssueBadRequestException(
                        errorBody = parseCreateIssueError(rawBody),
                        rawBody = rawBody,
                    )

                } else {

                    JiraException(
                        status = BAD_REQUEST,
                        errorCode = JIRA_INTEGRATION_ERROR,
                        message = rawBody.ifBlank {
                            "Bad request from Jira"
                        },
                    )
                }
            }

            else -> {

                JiraException(
                    status = SERVICE_UNAVAILABLE,
                    errorCode = response.status().toString(),
                    message = buildExternalErrorMessage(rawBody),
                )
            }
        }
    }

    private fun readResponseBody(
        response: Response
    ): String {

        return try {

            response.body()
                ?.asInputStream()
                ?.bufferedReader(Charsets.UTF_8)
                ?.use { it.readText() }
                .orEmpty()

        } catch (exception: Exception) {

            ""
        }
    }

    private fun parseCreateIssueError(
        rawBody: String
    ): CreateIssueErrorDto {

        return runCatching {

            objectMapper.readValue(
                rawBody,
                CreateIssueErrorDto::class.java,
            )

        }.getOrElse {

            CreateIssueErrorDto(
                errorMessages = listOfNotNull(
                    rawBody.takeIf { it.isNotBlank() }
                ),
                errors = emptyMap(),
            )
        }
    }

    private fun buildExternalErrorMessage(
        rawBody: String
    ): String {

        if (rawBody.isBlank()) {
            return "Jira integration error"
        }

        val externalError =
            runCatching {
                objectMapper.readValue(
                    rawBody,
                    JiraExternalErrorDto::class.java,
                )
            }.getOrNull()

        val messageFromExternalError =
            listOfNotNull(
                externalError?.smileCode,
                externalError?.error,
            ).joinToString(separator = " ")

        if (messageFromExternalError.isNotBlank()) {
            return messageFromExternalError
        }

        val jiraError =
            runCatching {
                objectMapper.readValue(
                    rawBody,
                    JiraErrorDto::class.java,
                )
            }.getOrNull()

        val messageFromJiraError =
            listOf(
                jiraError?.errorMessages.orEmpty().joinToString(separator = "; "),
                jiraError?.errors.orEmpty().values.joinToString(separator = "; "),
            )
                .filter { it.isNotBlank() }
                .joinToString(separator = "; ")

        return messageFromJiraError.ifBlank {
            rawBody
        }
    }

    private fun isCreateIssueMethod(
        methodKey: String
    ): Boolean {
        return methodKey.contains("JiraFeignClient#createIssue")
    }
}
```
