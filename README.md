```java
class JiraFeignErrorDecoderTest {

    private val objectMapper = jacksonObjectMapper()

    private val messageProvider = mockk<MessageProvider>()

    private lateinit var decoder: JiraFeignErrorDecoder

    @BeforeEach
    fun setUp() {
        decoder = JiraFeignErrorDecoder(
            objectMapper = objectMapper,
            messageProvider = messageProvider,
        )
    }

    @Test
    fun `decode should return RetryableException when jira returns 429`() {
        every {
            messageProvider["sbp.toomanyrequests.error"]
        } returns "Too many requests"

        val response = buildResponse(
            status = HttpStatus.TOO_MANY_REQUESTS,
            body = """
                {
                  "message": "too many requests"
                }
            """.trimIndent(),
        )

        val exception = decoder.decode(
            methodKey = METHOD_KEY,
            response = response,
        )

        assertThat(exception).isInstanceOf(RetryableException::class.java)

        val retryableException = exception as RetryableException

        assertThat(retryableException.status())
            .isEqualTo(HttpStatus.TOO_MANY_REQUESTS.value())

        assertThat(retryableException.message)
            .contains("Too many requests")
    }

    @Test
    fun `decode should return CreateIssueBadRequestException when jira returns 400 with valid error body`() {
        val responseBody = """
            {
              "errorMessages": [
                "summary is required"
              ],
              "errors": {
                "summary": "summary is required"
              }
            }
        """.trimIndent()

        val response = buildResponse(
            status = HttpStatus.BAD_REQUEST,
            body = responseBody,
        )

        val exception = decoder.decode(
            methodKey = METHOD_KEY,
            response = response,
        )

        assertThat(exception)
            .isInstanceOf(CreateIssueBadRequestException::class.java)

        val badRequestException = exception as CreateIssueBadRequestException

        assertThat(badRequestException.rawBody)
            .isEqualTo(responseBody)

        assertThat(badRequestException.errorBody.errorMessages)
            .containsExactly("summary is required")

        assertThat(badRequestException.errorBody.errors)
            .containsEntry("summary", "summary is required")
    }

    @Test
    fun `decode should return CreateIssueBadRequestException with fallback body when jira returns 400 with invalid json`() {
        val responseBody = "not valid json"

        val response = buildResponse(
            status = HttpStatus.BAD_REQUEST,
            body = responseBody,
        )

        val exception = decoder.decode(
            methodKey = METHOD_KEY,
            response = response,
        )

        assertThat(exception)
            .isInstanceOf(CreateIssueBadRequestException::class.java)

        val badRequestException = exception as CreateIssueBadRequestException

        assertThat(badRequestException.rawBody)
            .isEqualTo(responseBody)

        assertThat(badRequestException.errorBody.errorMessages)
            .containsExactly(responseBody)

        assertThat(badRequestException.errorBody.errors)
            .isEmpty()
    }

    @Test
    fun `decode should return CreateIssueBadRequestException with fallback message when jira returns 400 without body`() {
        val response = buildResponse(
            status = HttpStatus.BAD_REQUEST,
            body = "",
        )

        val exception = decoder.decode(
            methodKey = METHOD_KEY,
            response = response,
        )

        assertThat(exception)
            .isInstanceOf(CreateIssueBadRequestException::class.java)

        val badRequestException = exception as CreateIssueBadRequestException

        assertThat(badRequestException.rawBody)
            .isEqualTo("Jira returned bad request without response body")

        assertThat(badRequestException.errorBody.errorMessages)
            .containsExactly("Jira returned bad request without response body")

        assertThat(badRequestException.errorBody.errors)
            .isEmpty()
    }

    @Test
    fun `decode should return JiraException with service unavailable when jira returns 500`() {
        val responseBody = """
            {
              "error": "internal jira error"
            }
        """.trimIndent()

        val response = buildResponse(
            status = HttpStatus.INTERNAL_SERVER_ERROR,
            body = responseBody,
        )

        val exception = decoder.decode(
            methodKey = METHOD_KEY,
            response = response,
        )

        assertThat(exception)
            .isInstanceOf(JiraException::class.java)

        val jiraException = exception as JiraException

        assertThat(jiraException.status)
            .isEqualTo(HttpStatus.SERVICE_UNAVAILABLE)

        assertThat(jiraException.errorCode)
            .isEqualTo(HttpStatus.INTERNAL_SERVER_ERROR.value().toString())

        assertThat(jiraException.message)
            .isEqualTo(responseBody)
    }

    @Test
    fun `decode should return JiraException with fallback message when jira returns 503 without body`() {
        val response = buildResponse(
            status = HttpStatus.SERVICE_UNAVAILABLE,
            body = "",
        )

        val exception = decoder.decode(
            methodKey = METHOD_KEY,
            response = response,
        )

        assertThat(exception)
            .isInstanceOf(JiraException::class.java)

        val jiraException = exception as JiraException

        assertThat(jiraException.status)
            .isEqualTo(HttpStatus.SERVICE_UNAVAILABLE)

        assertThat(jiraException.errorCode)
            .isEqualTo(HttpStatus.SERVICE_UNAVAILABLE.value().toString())

        assertThat(jiraException.message)
            .isEqualTo("Jira returned error without response body. Status: 503")
    }

    private fun buildResponse(
        status: HttpStatus,
        body: String,
    ): Response {
        return Response.builder()
            .status(status.value())
            .reason(status.reasonPhrase)
            .request(buildRequest())
            .body(body, StandardCharsets.UTF_8)
            .build()
    }

    private fun buildRequest(): Request {
        return Request.create(
            Request.HttpMethod.POST,
            "/rest/api/2/issue",
            emptyMap(),
            ByteArray(0),
            StandardCharsets.UTF_8,
            null,
        )
    }

    private companion object {

        const val METHOD_KEY = "JiraFeignClient#createIssue"
    }
}
```
