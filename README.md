```java

SELECT
    ji.jira_key AS crossgoal,
    a.id AS initiative_id,
    a.agent_name
FROM jira_issue ji
JOIN ai_agent a ON a.id = ji.agent_id
WHERE LOWER(ji.project) = 'crossgoal'
  AND ji.type = 'initiative'
  AND ji.jira_key IS NOT NULL
  AND TRIM(ji.jira_key) <> ''
  AND a.disabled = false
  AND NOT EXISTS (
      SELECT 1
      FROM initiative_meta_ai_object obj
      WHERE obj.initiative_id = a.id
  )
ORDER BY a.id DESC
LIMIT 30;
```
