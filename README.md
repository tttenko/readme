```java

import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.assertThrows

class JiraUpdateSearchRequestFactoryTest {

    private val factory = JiraUpdateSearchRequestFactory()

    // --- createUpdatedTasksRequest ---

    @Test
    fun `updated tasks request should pass maxResults and startAt`() {
        val request = factory.createUpdatedTasksRequest(updateDepth = 7, maxResults = 100, startAt = 50)

        assertEquals(100, request.maxResults)
        assertEquals(50, request.startAt)
    }

    @Test
    fun `updated tasks request should contain correct jql`() {
        val request = factory.createUpdatedTasksRequest(updateDepth = 2, maxResults = 100, startAt = 0)

        val expectedJql = "project=CROSSGOAL AND issuetype=Task " +
            "AND updated>-2d " +
            "AND \"Epic Link\" IN (\"Мониторинг портфеля AI-Native\")"

        assertEquals(expectedJql, request.jql)
    }

    @Test
    fun `updated tasks request should contain full task fields set`() {
        val request = factory.createUpdatedTasksRequest(updateDepth = 7, maxResults = 100, startAt = 0)

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

        assertEquals(expectedFields, request.fields)
    }

    @Test
    fun `updated tasks request should fail when update depth is not positive`() {
        assertThrows<IllegalArgumentException> {
            factory.createUpdatedTasksRequest(updateDepth = 0, maxResults = 100, startAt = 0)
        }

        assertThrows<IllegalArgumentException> {
            factory.createUpdatedTasksRequest(updateDepth = -1, maxResults = 100, startAt = 0)
        }
    }

    @Test
    fun `updated tasks request should fail when maxResults is not positive`() {
        assertThrows<IllegalArgumentException> {
            factory.createUpdatedTasksRequest(updateDepth = 7, maxResults = 0, startAt = 0)
        }

        assertThrows<IllegalArgumentException> {
            factory.createUpdatedTasksRequest(updateDepth = 7, maxResults = -1, startAt = 0)
        }
    }

    // --- createUpdatedInitiativesRequest ---

    @Test
    fun `updated initiatives request should pass maxResults and startAt`() {
        val request = factory.createUpdatedInitiativesRequest(updateDepth = 7, maxResults = 100, startAt = 50)

        assertEquals(100, request.maxResults)
        assertEquals(50, request.startAt)
    }

    @Test
    fun `updated initiatives request should contain correct jql`() {
        val request = factory.createUpdatedInitiativesRequest(updateDepth = 2, maxResults = 100, startAt = 0)

        val expectedJql = "project=CROSSGOAL AND issuetype=Инициатива " +
            "AND updated>-2d " +
            "AND ((resolution=Unresolved AND labels IN (AI_Native_портфель,\"AI-эффективность\")) " +
            "OR labels=ClassicML OR status=\"Отменено\")"

        assertEquals(expectedJql, request.jql)
    }

    @Test
    fun `updated initiatives request should contain full initiative fields set`() {
        val request = factory.createUpdatedInitiativesRequest(updateDepth = 7, maxResults = 100, startAt = 0)

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

        assertEquals(expectedFields, request.fields)
    }

    @Test
    fun `updated initiatives request should fail when update depth is not positive`() {
        assertThrows<IllegalArgumentException> {
            factory.createUpdatedInitiativesRequest(updateDepth = 0, maxResults = 100, startAt = 0)
        }

        assertThrows<IllegalArgumentException> {
            factory.createUpdatedInitiativesRequest(updateDepth = -1, maxResults = 100, startAt = 0)
        }
    }

    @Test
    fun `updated initiatives request should fail when maxResults is not positive`() {
        assertThrows<IllegalArgumentException> {
            factory.createUpdatedInitiativesRequest(updateDepth = 7, maxResults = 0, startAt = 0)
        }

        assertThrows<IllegalArgumentException> {
            factory.createUpdatedInitiativesRequest(updateDepth = 7, maxResults = -1, startAt = 0)
        }
    }
}
```
