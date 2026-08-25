```java

val currentStatus =
    referenceData.statusesByCode[currentStatusCode]
        ?: throw IllegalStateException(
            "Calculated initiative status was not found in reference data: " +
                "jiraKey=$initiativeJiraKey, " +
                "agentId=${agent.id}, " +
                "statusCode=$currentStatusCode"
        )

```
