```java


SELECT
    COUNT(DISTINCT a.id) AS initiatives_count
FROM jira_issue ji
JOIN ai_agent a ON a.id = ji.agent_id
WHERE LOWER(ji.project) = 'crossgoal'
  AND ji.type = 'initiative'
  AND ji.jira_key = 'CROSSGOAL-41438'
  AND a.disabled = false;
```
