```java
SLA по выбранным инициативам (id = 3, 6, 545, 576, 699):
SELECT
    ass.ai_agent_id,
    a.agent_id,
    a.agent_name,
    a.disabled,
    current_status.code AS current_status,
    current_status.ordering AS current_ordering,
    sla_status.code AS stage_code,
    sla_status.ordering AS stage_ordering,
    ass.planned_date,
    ass.completed_date
FROM agent_status_sla ass
JOIN ai_agent a
    ON a.id = ass.ai_agent_id
LEFT JOIN status current_status
    ON current_status.id = a.agent_status_id
JOIN status sla_status
    ON sla_status.id = ass.agent_status_id
WHERE ass.ai_agent_id IN (
    3,
    6,
    545,
    576,
    699
)
ORDER BY ass.ai_agent_id, sla_status.ordering;
Ожидаемые статусы этапов по бизнес-логике:
WITH initiatives AS (
    SELECT
        a.id,
        a.agent_id,
        a.agent_name,
        a.disabled,
        s.code AS raw_status_code,
        CASE s.code
            WHEN 'research' THEN 'analysis'
            WHEN 'release' THEN 'targetSolution'
            ELSE s.code
        END AS showcase_status_code
    FROM ai_agent a
    LEFT JOIN status s
        ON s.id = a.agent_status_id
    WHERE a.id IN (
        3,
        6,
        545,
        576,
        699
    )
),
current_status AS (
    SELECT
        i.*,
        s.ordering AS current_ordering
    FROM initiatives i
    LEFT JOIN status s
        ON s.code = i.showcase_status_code
       AND (s.disabled = false OR s.disabled IS NULL)
),
stages AS (
    SELECT
        id,
        code,
        ordering
    FROM status
    WHERE code IN (
        'analysis',
        'development',
        'pilot',
        'targetSolution'
    )
      AND (disabled = false OR disabled IS NULL)
)
SELECT
    cs.id,
    cs.agent_id,
    cs.agent_name,
    cs.disabled,
    cs.raw_status_code,
    cs.showcase_status_code,
    stage.code AS stage_code,
    ass.planned_date,
    ass.completed_date,
    CASE
        WHEN ass.completed_date IS NOT NULL
            THEN 'Завершён'
        WHEN cs.current_ordering IS NULL
            THEN 'План'
        WHEN stage.ordering < cs.current_ordering
            THEN 'Завершён'
        WHEN stage.ordering = cs.current_ordering
            THEN 'В работе'
        ELSE 'План'
    END AS expected_stage_status
FROM current_status cs
CROSS JOIN stages stage
LEFT JOIN agent_status_sla ass
    ON ass.ai_agent_id = cs.id
   AND ass.agent_status_id = stage.id
ORDER BY cs.id, stage.ordering;
Контакты/email для тех же инициатив:
SELECT
    a.id,
    a.agent_id,
    a.agent_name,
    STRING_AGG(c.email, ';') AS expected_contact_emails
FROM ai_agent a
LEFT JOIN agent_contact ac
    ON ac.agent_id = a.id
LEFT JOIN contact c
    ON c.id = ac.contact_id
   AND c.email IS NOT NULL
   AND BTRIM(c.email) <> ''
WHERE a.id IN (
    3,
    6,
    545,
    576,
    699
)
GROUP BY
    a.id,
    a.agent_id,
    a.agent_name
ORDER BY a.id;
```
