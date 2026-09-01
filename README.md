```java

select
    id,
    agent_id,
    agent_name,
    agent_jira_url,
    block_id,
    division_id
from ai_agent
where agent_id = 'CROSSGOAL-150657'
   or agent_jira_url ilike '%CROSSGOAL-150657%'
   or agent_name = 'Новая инициатива 31';

```
