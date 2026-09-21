```java
select
    md.id                         as metric_id,
    md.name                       as metric_name,
    md.active                     as metric_active,
    md.frequency,
    md.copilot_applicability,

    ima.id                        as assignment_id,
    coalesce(
        ima.applicability_status,
        'ACTIVE'
    )                             as expected_applicability_status,

    mar.id                        as latest_request_id,
    mar.status                    as latest_request_status,
    mar.resume_period             as expected_resume_period,
    mar.created_at                as request_created_at

from metrics_directory md

left join initiative_metric_assignment ima
    on ima.metric_directory_id = md.id
   and ima.initiative_agent_type_id = 2

left join lateral (
    select r.*
    from metric_applicability_request r
    where r.initiative_metric_assignment_id = ima.id
    order by r.created_at desc, r.id desc
    limit 1
) mar on true

where md.copilot_applicability = true

order by md.name;


```
