```java
select
    ass.ai_agent_id,
    s.code,
    ass.planned_date,
    ass.completed_date
from agent_status_sla ass
join status s
    on s.id = ass.agent_status_id
where ass.ai_agent_id = 545;
```
