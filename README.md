```java
 Шаг 1. Найдём инициативу с типами агентов

Выполни:

select
    imt.ai_agent_id          as initiative_id,
    a.agent_name             as initiative_name,
    imt.id                   as initiative_agent_type_id,
    imt.agent_type
from initiative_metric_type imt
join ai_agent a
    on a.id = imt.ai_agent_id
order by imt.ai_agent_id, imt.agent_type;

Лучше выбрать инициативу, у которой есть хотя бы autonomous или copilot, а идеально — оба.

Шаг 2. Для выбранной инициативы посмотрим assignments

После того как выберешь initiative_id, подставь его:

select
    imt.ai_agent_id              as initiative_id,
    imt.id                       as initiative_agent_type_id,
    imt.agent_type,
    ima.id                       as assignment_id,
    ima.metric_directory_id      as metric_id,
    md.name                      as metric_name,
    ima.applicability_status
from initiative_metric_type imt
left join initiative_metric_assignment ima
    on ima.initiative_agent_type_id = imt.id
left join metrics_directory md
    on md.id = ima.metric_directory_id
where imt.ai_agent_id = <INITIATIVE_ID>
order by imt.agent_type, md.name;

Нам особенно интересны строки с:

ACTIVE
PENDING
NOT_APPLICABLE

Но отсутствие assignment тоже важно: по требованиям оно должно интерпретироваться как ACTIVE.

Шаг 3. Посмотрим последнюю заявку для каждого assignment
select
    ima.id                    as assignment_id,
    imt.ai_agent_id           as initiative_id,
    imt.agent_type,
    ima.metric_directory_id   as metric_id,
    md.name                   as metric_name,
    ima.applicability_status,
    mar.id                    as request_id,
    mar.status                as request_status,
    mar.resume_period,
    mar.created_at
from initiative_metric_assignment ima
join initiative_metric_type imt
    on imt.id = ima.initiative_agent_type_id
join metrics_directory md
    on md.id = ima.metric_directory_id
left join lateral (
    select r.*
    from metric_applicability_request r
    where r.initiative_metric_assignment_id = ima.id
    order by r.created_at desc, r.id desc
    limit 1
) mar on true
where imt.ai_agent_id = <INITIATIVE_ID>
order by imt.agent_type, md.name;

Именно created_at DESC, id DESC нам сейчас особенно важно проверить, потому что последняя заявка используется для resumePeriod.


```
