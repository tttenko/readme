```java

-- 1. Есть ли стратегия для Jira key?
select id, name, jira_issue
from strategy
where upper(jira_issue) like '%CROSSGOAL-55639%';

-- 2. Что за агент 2208?
select *
from ai_agent
where id = 2208;

-- 3. Какие стратегии вообще есть у агента?
select *
from agent_strategy
where agent_id = 2208;

```
