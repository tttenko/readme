```java

Сначала можешь проверить текущих активных:

SELECT id,
       agent_id,
       agent_name,
       agent_jira_url,
       disabled
FROM ai_agent
WHERE agent_id NOT LIKE '%PULT%'
  AND agent_jira_url LIKE '%CROSSGOAL-%'
  AND disabled != true
ORDER BY id;

Если их действительно 2, включи ещё 18:

UPDATE ai_agent
SET disabled = false
WHERE id IN (
    SELECT id
    FROM ai_agent
    WHERE agent_id NOT LIKE '%PULT%'
      AND agent_jira_url LIKE '%CROSSGOAL-%'
      AND disabled = true
    ORDER BY id
    LIMIT 18
);

После этого проверь:

SELECT COUNT(*)
FROM ai_agent
WHERE agent_id NOT LIKE '%PULT%'
  AND agent_jira_url LIKE '%CROSSGOAL-%'
  AND disabled != true;

Должно вернуть:

20
```
