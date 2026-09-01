```java
-- 1. agent_quality_gate
select
    ai_agent_id,
    quality_gate_code,
    state,
    created,
    updated
from agent_quality_gate
where ai_agent_id = 739
order by quality_gate_code;


-- 2. jira_issue
select
    id,
    agent_id,
    project,
    type,
    jira_id,
    jira_key,
    jira_url,
    parent_id,
    quality_gate_code,
    jira_link,
    jira_transition,
    created,
    updated
from jira_issue
where agent_id = 739
order by id;

По логам ожидается:

в agent_quality_gate — 25 записей со state = 'unchecked';
в jira_issue — 2 записи:
CROSSGOAL-999901;
GIGAUSAGE-7631.

Можно сразу проверить количества:

select count(*)
from agent_quality_gate
where ai_agent_id = 739;

select count(*)
from jira_issue
where agent_id = 739;

Ожидаемо:

agent_quality_gate = 25
jira_issue = 2

И более строгая проверка jira_issue:

select jira_key, project, type
from jira_issue
where agent_id = 739
  and jira_key in (
      'CROSSGOAL-999901',
      'GIGAUSAGE-7631'
  );
```
