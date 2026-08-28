```java

class JiraInitiativeSearchRequestFactoryTest {

    private val factory = JiraInitiativeSearchRequestFactory()

    // --- createNewInitiativesRequest ---

    @Test
    fun `new initiatives request should pass maxResults and startAt`() {
        val request = factory.createNewInitiativesRequest(newDepth = 7, maxResults = 100, startAt = 50)

        assert(request.maxResults == 100)
        assert(request.startAt == 50)
    }

    @Test
    fun `new initiatives request should embed newDepth in jql`() {
        val request = factory.createNewInitiativesRequest(newDepth = 30, maxResults = 100, startAt = 0)

        assert(request.jql.contains("created>-30"))
    }

    @Test
    fun `new initiatives request should contain correct jql`() {
        val request = factory.createNewInitiativesRequest(newDepth = 7, maxResults = 100, startAt = 0)

        val expectedJql =
            "project=CROSSGOAL " +
                "AND issuetype=Инициатива " +
                "AND resolution=Unresolved " +
                "AND labels IN (AI_Native_портфель, AI-эффективность) " +
                "AND created>-7"

        assert(request.jql == expectedJql)
    }

    @Test
    fun `new initiatives request should not contain order by`() {
        val request = factory.createNewInitiativesRequest(newDepth = 7, maxResults = 100, startAt = 0)

        assert(!request.jql.contains("ORDER BY", ignoreCase = true))
    }

    @Test
    fun `new initiatives request jql should be single line`() {
        val request = factory.createNewInitiativesRequest(newDepth = 7, maxResults = 100, startAt = 0)

        assert(!request.jql.contains("\n"))
        assert(!request.jql.contains("\r"))
    }

    @Test
    fun `new initiatives request should contain full initiative fields set`() {
        val request = factory.createNewInitiativesRequest(newDepth = 7, maxResults = 100, startAt = 0)

        val expectedFields = listOf(
            "summary",
            "description",
            "status",
            "labels",
            "customfield_30000",
            "customfield_30001",
            "customfield_30002",
            "customfield_34300",
            "customfield_30401",
            "customfield_31304",
            "customfield_31305",
            "customfield_31306",
            "customfield_31307",
            "issuelinks",
            "customfield_15903",
            "assignee",
            "reporter",
            "customfield_29202",
            "customfield_29203",
            "customfield_29205",
            "lastViewed",
            "resolutiondate",
            "created",
            "updated",
        )

        assert(request.fields == expectedFields)
    }

    // --- createMonitoringTasksRequest ---

    @Test
    fun `monitoring tasks request should pass maxResults and startAt`() {
        val request = factory.createMonitoringTasksRequest(epicKey = "CROSSGOAL-100", maxResults = 50, startAt = 0)

        assert(request.maxResults == 50)
        assert(request.startAt == 0)
    }

    @Test
    fun `monitoring tasks request should embed epic key in jql`() {
        val request = factory.createMonitoringTasksRequest(epicKey = "CROSSGOAL-200", maxResults = 50, startAt = 0)

        assert(request.jql.contains("project = CROSSGOAL"))
        assert(request.jql.contains("\"Epic Link\" = CROSSGOAL-200"))
    }

    @Test
    fun `monitoring tasks request jql should be single line`() {
        val request = factory.createMonitoringTasksRequest(epicKey = "CROSSGOAL-100", maxResults = 50, startAt = 0)

        assert(!request.jql.contains("\n"))
        assert(!request.jql.contains("\r"))
    }

    @Test
    fun `monitoring tasks request should contain full monitoring task fields set`() {
        val request = factory.createMonitoringTasksRequest(epicKey = "CROSSGOAL-100", maxResults = 50, startAt = 0)

        val expectedFields = listOf(
            "summary",
            "description",
            "status",
            "customfield_16700",
            "customfield_16701",
            "assignee",
            "reporter",
            "lastViewed",
            "resolutiondate",
            "created",
            "updated",
        )

        assert(request.fields == expectedFields)
    }
}
```
