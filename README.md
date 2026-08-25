```java

fun findMonitoringEpic(
    initiativeJiraKey: String,
    issueLinks: List<GetIssueLinkResponse>?,
): JiraMonitoringEpicData? {

    return issueLinks
        .orEmpty()
        .asSequence()
        .flatMap { issueLink ->
            sequenceOf(
                issueLink.outwardIssue,
                issueLink.inwardIssue,
            )
        }
        .filterNotNull()
        .filter { linkedIssue ->
            linkedIssue.fields
                ?.summary
                ?.contains(
                    other = MONITORING_SUMMARY_PART,
                    ignoreCase = true,
                ) == true
        }
        .mapNotNull { linkedIssue ->
            val epicKey =
                jiraIssueKeyExtractor.extractCrossgoalKey(
                    linkedIssue.key
                )

            if (epicKey == null) {
                log.warn(
                    "Skipping monitoring epic candidate with invalid CROSSGOAL key: " +
                        "initiativeKey={}, epicKey={}",
                    initiativeJiraKey,
                    linkedIssue.key,
                )

                return@mapNotNull null
            }

            JiraMonitoringEpicData(
                jiraId = linkedIssue.id,
                jiraKey = epicKey,
            )
        }
        .firstOrNull()
}
```
