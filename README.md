```java

UPDATE ai_agent
SET disabled = true
WHERE id IN (
    SELECT id
    FROM ai_agent
    WHERE agent_id NOT LIKE '%PULT%'
      AND agent_jira_url LIKE '%CROSSGOAL-%'
      AND disabled != true
    ORDER BY id DESC
    LIMIT 30
);

Потом проверь:

SELECT COUNT(*)
FROM ai_agent
WHERE agent_id NOT LIKE '%PULT%'
  AND agent_jira_url LIKE '%CROSSGOAL-%'
  AND disabled != true;
```
