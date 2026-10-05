```java

select task.agent_id,
       agent.agent_id as initiative_code,
       agent.disabled,
       task.id as task_relation_id,
       task.parent_id,
       epic.jira_key as monitoring_epic,
       task.created
from jira_issue task
join ai_agent agent on agent.id = task.agent_id
left join jira_issue epic on epic.id = task.parent_id
where task.jira_key = 'CROSSGOAL-998910'
  and task.type = 'task'
  and lower(task.project) = 'crossgoal'
order by task.created desc, task.agent_id;

select *
from quality_gate
where code in (
    'analysis', 'development', 'pilot',
    'targetSolution', 'feedback'
)
order by code;

select sla.agent_status_id,
       status.code as status_code,
       sla.planned_date,
       sla.completed_date
from agent_status_sla sla
left join status on status.id = sla.agent_status_id
where sla.ai_agent_id = 740
order by status.code;

```
