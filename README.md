```java


SELECT
    COUNT(DISTINCT a.id) AS initiatives_count
FROM jira_issue ji
JOIN ai_agent a ON a.id = ji.agent_id
WHERE LOWER(ji.project) = 'crossgoal'
  AND ji.type = 'initiative'
  AND ji.jira_key = 'CROSSGOAL-41438'
  AND a.disabled = false;

  SELECT
    COUNT(*) AS history_count,
    SUM(changed_objects_count) AS changed_objects_count
FROM initiative_ai_object_history
WHERE file_upload_id = 119;
```
