```java

SELECT COUNT(*)
FROM ai_agent
WHERE agent_id NOT LIKE '%PULT%'
  AND agent_jira_url LIKE '%CROSSGOAL-%'
  AND disabled != true;

И посмотри несколько подходящих агентов:

SELECT id,
       agent_id,
       agent_name,
       agent_jira_url,
       disabled
FROM ai_agent
WHERE agent_id NOT LIKE '%PULT%'
  AND agent_jira_url LIKE '%CROSSGOAL-%'
  AND disabled != true
ORDER BY id
LIMIT 20;

Выбери, например, два id.

Допустим:

123
456

Тогда выключи только тех агентов, которые участвуют в этом scheduler, а не вообще всех агентов системы:

UPDATE ai_agent
SET disabled = true
WHERE agent_id NOT LIKE '%PULT%'
  AND agent_jira_url LIKE '%CROSSGOAL-%';

И включи обратно только двух:

UPDATE ai_agent
SET disabled = false
WHERE id IN (123, 456);

После этого проверка:

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

В идеале здесь останутся ровно 2 строки.

```
