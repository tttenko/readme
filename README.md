```java
select
    imt.ai_agent_id as initiative_id,
    imt.agent_type,
    md.id as metric_id,
    md.name as metric_name,
    md.active,
    ima.id as assignment_id,
    ima.applicability_status
from initiative_metric_type imt
cross join metrics_directory md
left join initiative_metric_assignment ima
    on ima.initiative_agent_type_id = imt.id
    and ima.metric_directory_id = md.id
where md.active = true
  and (
      ima.id is null
      or ima.applicability_status = 'ACTIVE'
  )
order by imt.ai_agent_id, imt.agent_type
limit 20;
```
