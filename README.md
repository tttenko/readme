```java

@Test
fun `updated tasks request should contain correct jql`() {
    val request = factory.createUpdatedTasksRequest(updateDepth = 2, maxResults = 100, startAt = 0)

    val expectedJql = "project=CROSSGOAL AND issuetype=Task " +
        "AND updated>-2d " +
        "AND cf[10006]='Мониторинг портфеля AI-Native'"

    assertEquals(expectedJql, request.jql)
}

@Test
fun `updated initiatives request should contain correct jql`() {
    val request = factory.createUpdatedInitiativesRequest(updateDepth = 2, maxResults = 100, startAt = 0)

    val expectedJql = "project=CROSSGOAL AND issuetype=Инициатива " +
        "AND updated>-2d " +
        "AND ((resolution=Unresolved AND labels IN (AI_Native_портфель,'AI-эффективность')) " +
        "OR labels=ClassicML OR status='Отменено')"

    assertEquals(expectedJql, request.jql)
}

private fun assertSearchContract() {
    val taskRequest = jiraStub.searchRequests(Fr2SearchKind.UPDATED_TASKS).first()
    val initiativeRequest = jiraStub.searchRequests(Fr2SearchKind.INITIATIVES).first()

    val expectedTaskJql = "project=CROSSGOAL AND issuetype=Task " +
        "AND updated>-3d " +
        "AND cf[10006]='Мониторинг портфеля AI-Native'"

    assertThat(taskRequest.maxResults).isEqualTo(PAGE_SIZE)
    assertThat(taskRequest.jql).isEqualTo(expectedTaskJql)
    assertThat(taskRequest.fields).contains("status", "customfield_16701", "resolutiondate")

    val expectedInitiativeJql = "project=CROSSGOAL AND issuetype=Инициатива " +
        "AND updated>-3d " +
        "AND ((resolution=Unresolved AND labels IN (AI_Native_портфель,'AI-эффективность')) " +
        "OR labels=ClassicML OR status='Отменено')"

    assertThat(initiativeRequest.maxResults).isEqualTo(PAGE_SIZE)
    assertThat(initiativeRequest.jql).isEqualTo(expectedInitiativeJql)
    assertThat(initiativeRequest.fields).contains("summary", "status", "labels", "issuelinks", "updated", "created")
}
```
