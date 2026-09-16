```java

select
    a.id as initiative_id,
    a.agent_name,
    imt.id as initiative_metric_type_id,
    imt.agent_type
from ai_agent a
left join initiative_metric_type imt on imt.ai_agent_id = a.id
where a.id = 37;

Например:

INSERT INTO initiative_metric_assignment (
    initiative_agent_type_id,
    metric_id,
    applicability_status
)
VALUES (
    37,
    '055b6470-af1e-41cd-a912-3440f3d7d3e5',
    'NOT_APPLICABLE'
)
RETURNING id;

Получил, допустим:

id = 101

Дальше создаём request:

INSERT INTO metric_applicability_request (
    initiative_metric_assignment_id,
    status,
    comment,
    resume_period,
    is_visible_in_office,
    effective_from_period,
    effective_to_period,
    created_by,
    decision_by,
    created_at,
    updated_at
)
VALUES (
    101,
    'APPROVED',
    'TEST_HISTORY_REQUEST',
    DATE '2026-12-01',
    true,
    DATE '2026-10-01',
    DATE '2026-12-01',
    123456,
    123456,
    DATE '2026-09-15',
    TIMESTAMP '2026-09-15 12:00:00'
)
RETURNING id;

Допустим получили:

id = 201

Теперь history:

INSERT INTO metric_applicability_history (
    metric_applicability_request_id,
    action,
    created_by,
    comment,
    created_at
)
VALUES (
    201,
    'REQUEST_CREATED',
    '123456',
    'TEST_HISTORY_REQUEST_CREATED',
    DATE '2026-09-15'
);

И:

INSERT INTO metric_applicability_history (
    metric_applicability_request_id,
    action,
    created_by,
    comment,
    created_at
)
VALUES (
    201,
    'APPROVED',
    '123456',
    'TEST_HISTORY_APPROVED',
    DATE '2026-09-15'
);

Для проверки SYSTEM:

INSERT INTO metric_applicability_history (
    metric_applicability_request_id,
    action,
    created_by,
    comment,
    created_at
)
VALUES (
    201,
    'RESUME_PERIOD',
    'SYSTEM',
    'TEST_HISTORY_RESUME_PERIOD',
    DATE '2026-12-01'
);
```
