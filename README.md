```java
select
    id,
    agent_name,
    agent_description,
    agent_problem,
    agent_effect_revenue,
    agent_effect_optimization,
    import_status,
    jira_status
from ai_agent
where id = 545;

select
    jira_key,
    jira_url,
    project,
    type
from jira_issue
where agent_id = 545
  and project = 'gigausage'
  and type = 'initiative';

select
    change_type,
    payload,
    created
from jira_change
where agent_id = 545
order by id desc
limit 2;

select *
from agent_enabler
where agent_id = 545;

  
```
