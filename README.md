```java

private fun assertSearchContract() {
    val taskRequest = jiraStub.searchRequests(Fr2SearchKind.UPDATED_TASKS).first()
    val initiativeRequest = jiraStub.searchRequests(Fr2SearchKind.INITIATIVES).first()

    val expectedTaskJql = "project=CROSSGOAL AND issuetype=Task " +
        "AND updated>-3d " +
        "AND \"Epic Link\" IN (\"Мониторинг портфеля AI-Native\")"

    assertThat(taskRequest.maxResults).isEqualTo(PAGE_SIZE)
    assertThat(taskRequest.jql).isEqualTo(expectedTaskJql)
    assertThat(taskRequest.fields).contains("status", "customfield_16701", "resolutiondate")

    val expectedInitiativeJql = "project=CROSSGOAL AND issuetype=Инициатива " +
        "AND updated>-3d " +
        "AND ((resolution=Unresolved AND labels IN (AI_Native_портфель,\"AI-эффективность\")) " +
        "OR labels=ClassicML OR status=\"Отменено\")"

    assertThat(initiativeRequest.maxResults).isEqualTo(PAGE_SIZE)
    assertThat(initiativeRequest.jql).isEqualTo(expectedInitiativeJql)
    assertThat(initiativeRequest.fields).contains("summary", "status", "labels", "issuelinks", "updated", "created")
}
```
