```java

    private val objectMapper = jacksonObjectMapper()
    private val searchCalls = CopyOnWriteArrayList<Pair<Fr2SearchKind, SearchIssueRequestDto>>()
    private val issueCalls = CopyOnWriteArrayList<String>()
    private val issues = ConcurrentHashMap<String, Map<String, Any>>()
    private val failingSearches = ConcurrentHashMap.newKeySet<Fr2SearchKind>()
    private val getFailuresRemaining = ConcurrentHashMap<String, AtomicInteger>()
    private val executor = Executors.newCachedThreadPool { runnable ->
        Thread(runnable, "jira-fr2-integration-stub").apply { isDaemon = true }
    }
    private val server = HttpServer.create(InetSocketAddress("localhost", 0), 0)

    init {
        server.createContext("/internal/v1/jira/search") { exchange -> handleSearch(exchange) }
        server.createContext("/internal/v1/jira/issue/") { exchange -> handleGet(exchange) }
        server.executor = executor
        server.start()
    }

    fun baseUrl() = "http://localhost:${server.address.port}"
    fun searchRequests(kind: Fr2SearchKind) = searchCalls.filter { it.first == kind }.map { it.second }
    fun issueRequests() = issueCalls.toList()
    fun failSearch(kind: Fr2SearchKind) { failingSearches += kind }
    fun failGet(issueKey: String, attempts: Int) { getFailuresRemaining[issueKey] = AtomicInteger(attempts) }

    fun putIssue(issueKey: String, statusId: String, statusName: String, labels: List<String> = emptyList()) {
        issues[issueKey] = mapOf("id" to issueKey.substringAfter('-'), "key" to issueKey,
            "fields" to mapOf("status" to mapOf("id" to statusId, "name" to statusName), "labels" to labels))
    }

    fun reset() {
        updatedTasks.clear()
        updatedInitiatives.clear()
        monitoringTasks.clear()
        searchCalls.clear()
        issueCalls.clear()
        issues.clear()
        failingSearches.clear()
        getFailuresRemaining.clear()
    }

    fun stop() {
        server.stop(0)
        executor.shutdownNow()
    }

    private fun handleSearch(exchange: HttpExchange) {
        try {
            if (exchange.requestMethod != "POST") return respond(exchange, 405, "Only POST")
            val request = objectMapper.readValue(exchange.requestBody, SearchIssueRequestDto::class.java)
            val kind = when {
                request.jql.contains("issuetype = Инициатива") -> Fr2SearchKind.INITIATIVES
                request.jql.contains("Мониторинг портфеля AI-Native") -> Fr2SearchKind.UPDATED_TASKS
                request.jql.contains("\"Epic Link\"") -> Fr2SearchKind.MONITORING
                else -> return respond(exchange, 500, "Unexpected JQL: ${request.jql}")
            }
            searchCalls += kind to request
            if (kind in failingSearches) return respond(exchange, 500, "Jira Search failed")
            val allIssues = when (kind) {
                Fr2SearchKind.UPDATED_TASKS -> updatedTasks
                Fr2SearchKind.INITIATIVES -> updatedInitiatives
                Fr2SearchKind.MONITORING -> monitoringTasks
            }
            val response = SearchIssueResponseDto(startAt = request.startAt, maxResults = request.maxResults,
                total = allIssues.size, issues = allIssues.drop(request.startAt ?: 0).take(request.maxResults))
            respond(exchange, 200, objectMapper.writeValueAsString(response))
        } catch (exception: Exception) {
            respond(exchange, 500, "Stub Search failed: ${exception.message}")
        }
    }

    private fun handleGet(exchange: HttpExchange) {
        try {
            if (exchange.requestMethod != "GET") return respond(exchange, 405, "Only GET")
            val issueKey = exchange.requestURI.path.substringAfterLast('/')
            issueCalls += issueKey
            if ((getFailuresRemaining[issueKey]?.getAndDecrement() ?: 0) > 0) {
                return respond(exchange, 500, "Jira GET failed: $issueKey")
            }
            val issue = issues[issueKey] ?: return respond(exchange, 404, "No Jira issue: $issueKey")
            respond(exchange, 200, objectMapper.writeValueAsString(issue))
        } catch (exception: Exception) {
            respond(exchange, 500, "Stub GET failed: ${exception.message}")
        }
    }

    private fun respond(exchange: HttpExchange, status: Int, body: String) {
        val bytes = body.toByteArray(StandardCharsets.UTF_8)
        exchange.responseHeaders.set("Content-Type", if (status == 200) "application/json" else "text/plain")
        exchange.sendResponseHeaders(status, bytes.size.toLong())
        exchange.responseBody.use { it.write(bytes) }
    }
}

private fun jiraDate(value: LocalDateTime): String =
    value.format(DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mm:ss.SSS")) + "+0300"

private const val PAGE_SIZE = 1
private const val MAIN_KEY = "CROSSGOAL-100"
private const val SECOND_KEY = "CROSSGOAL-200"
private const val ABSENT_KEY = "CROSSGOAL-300"
private const val NEW_EPIC_KEY = "CROSSGOAL-500"
private const val QG_TASK_KEY = "CROSSGOAL-501"
private const val ANALYSIS_TASK_KEY = "CROSSGOAL-502"
private const val DEVELOPMENT_TASK_KEY = "CROSSGOAL-503"
private const val QG_CODE = "QG_ARCHITECTURE_FR2"
private const val SECOND_QG_CODE = "QG_SECURITY_FR2"
private const val UPDATED_SUMMARY = "Updated FR2 initiative"
private const val SIGMA_URL_PREFIX = "http://jira.test/browse/"
```
