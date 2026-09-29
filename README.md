```java

select
    id,
    name
from metrics_directory
where name ilike '%test761%';

Если найдётся, возьми metric_id и выполни:
select
    mv.id as metric_value_id,
    mv.initiative_agent_type_id,
    mv.metric_directory_id,
    mv.period_month,
    mv.metric_value,
    mv.target_value,
    mt.ai_agent_id as initiative_id,
    mt.agent_type,
    a.agent_name
from initiative_metric_value mv
join initiative_metric_type mt
    on mt.id = mv.initiative_agent_type_id
join ai_agent a
    on a.id = mt.ai_agent_id
where mv.metric_directory_id = '<metric_id>'
order by mv.period_month desc;

```
