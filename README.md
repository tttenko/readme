```java
fun createUpdatedInitiativesRequest(updateDepth: Int, maxResults: Int, startAt: Int): SearchIssueRequestDto {
    require(updateDepth > 0) { "Jira updateDepth must be positive" }
    require(maxResults > 0) { "Jira maxResults must be positive" }

    return SearchIssueRequestDto(
        fields = INITIATIVE_FIELDS,
        jql = "project=CROSSGOAL AND issuetype=Инициатива " +
                "AND updated>-${updateDepth}d " +
                "AND ((resolution=Unresolved AND labels IN (AI_Native_портфель,'AI-эффективность')) " +
                "OR labels=ClassicML OR status='Отменена')",
        maxResults = maxResults,
        startAt = startAt
    )
}
```
