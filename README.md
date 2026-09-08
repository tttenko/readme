```java
(
    :#{#params.gigaUsageNotFilledEnabled} = true

    AND candidate.current_status_code IN (
        :#{#params.pilotStatus},
        :#{#params.targetSolutionStatus}
    )

    AND NOT EXISTS (
        SELECT 1
        FROM jira_issue jira
        WHERE jira.agent_id = candidate.id
          AND LOWER(jira.project) =
              LOWER(:#{#params.gigaUsageProject})
          AND NULLIF(
              BTRIM(jira.jira_key),
              ''
          ) IS NOT NULL
    )
)
```
