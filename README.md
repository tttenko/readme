```java

UPDATE ai_agent
SET disabled = false
WHERE id IN (
    SELECT id
    FROM ai_agent
    WHERE agent_id NOT LIKE '%PULT%'
      AND agent_jira_url LIKE '%CROSSGOAL-%'
      AND disabled = true
    ORDER BY id
    LIMIT 20
);

Проверка:

SELECT COUNT(*)
FROM ai_agent
WHERE agent_id NOT LIKE '%PULT%'
  AND agent_jira_url LIKE '%CROSSGOAL-%'
  AND disabled != true;

Должно вернуть:

20

И можно сразу посмотреть, какие именно 20 включились:

SELECT id, agent_id, agent_name, agent_jira_url, disabled
FROM ai_agent
WHERE agent_id NOT LIKE '%PULT%'
  AND agent_jira_url LIKE '%CROSSGOAL-%'
  AND disabled != true
ORDER BY id;
```
