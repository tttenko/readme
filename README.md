```java

select
    r.id,
    r.status,
    r.resume_period,
    r.is_visible_in_office,
    r.created_at,
    r.updated_at,
    r.effective_from_period,
    r.effective_to_period
from metric_applicability_request r
where r.initiative_metric_assignment_id = (
    select a.id
    from initiative_metric_assignment a
    join initiative_metric_type mt
        on mt.id = a.initiative_agent_type_id
    join metrics_directory md
        on md.id = a.metric_id
    where mt.ai_agent_id = <initiative_id>
      and mt.agent_type = '<agent_type>'
      and md.name = 'test761hMB'
)
order by r.created_at desc, r.id desc;
```
