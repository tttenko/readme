```java
select id, agent_id, agent_name, agent_jira_url
from ai_agent
where agent_id = 'CROSSGOAL-150657'
   or agent_jira_url ilike '%CROSSGOAL-150657%';

select *
from jira_issue
where jira_key = 'CROSSGOAL-150657';
```
