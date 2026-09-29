```java

select
    a.id as initiative_id,
    a.agent_name,
    mt.id as initiative_agent_type_id,
    mt.agent_type,
    md.id as metric_id,
    md.name as metric_name,
    assignment.id as assignment_id,
    assignment.applicability_status
from initiative_metric_assignment assignment
join initiative_metric_type mt
    on mt.id = assignment.initiative_agent_type_id
join ai_agent a
    on a.id = mt.ai_agent_id
join metrics_directory md
    on md.id = assignment.metric_id
where md.name = 'test761hMB';


```
