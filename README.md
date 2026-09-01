```java
-- Сам агент
select
    id,
    agent_id,
    block_id,
    division_id,
    agent_initiative_type,
    import_status,
    jira_from_status
from ai_agent
where id = 740;


-- Стратегия
select *
from agent_strategy
where ai_agent_id = 740;


-- Ресурсы
select *
from involved_resource
where ai_agent_id = 740;


-- Контакты
select
    ac.*,
    c.email,
    c.fio
from agent_contact ac
join contact c on c.id = ac.contact_id
where ac.agent_id = 740;


-- Enablers
select
    ae.*,
    e.name
from agent_enabler ae
join enabler e on e.id = ae.enabler_id
where ae.agent_id = 740;


-- Quality Gates
select *
from agent_quality_gate
where ai_agent_id = 740
order by quality_gate_code;


-- Jira relations
select
    id,
    agent_id,
    project,
    type,
    jira_id,
    jira_key,
    jira_url
from jira_issue
where agent_id = 740
order by id;
```
