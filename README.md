```java
Для агента 546 сначала проверь, что он существует и у него есть пройденные этапы:

SELECT
    a.id,
    a.agent_id,
    a.agent_name,
    s.id AS current_status_id,
    s.code AS current_status,
    s.ordering AS current_ordering
FROM prm_ai.ai_agent a
JOIN prm_ai.status s
    ON s.id = a.agent_status_id
WHERE a.id = 546;

Также проверь существующие SLA, чтобы случайно не затереть нужные данные:

SELECT
    s.code,
    s.ordering,
    sla.planned_date,
    sla.completed_date
FROM prm_ai.agent_status_sla sla
JOIN prm_ai.status s
    ON s.id = sla.agent_status_id
WHERE sla.ai_agent_id = 546
ORDER BY s.ordering;

Если это тестовый агент и данные можно заменить, выполни:

ROLLBACK;

DELETE FROM prm_ai.agent_status_sla
WHERE ai_agent_id = 546;

WITH current_stage AS (
    SELECT current_status.ordering AS current_ordering
    FROM prm_ai.ai_agent agent
    JOIN prm_ai.status current_status
        ON current_status.id = agent.agent_status_id
    WHERE agent.id = 546
),
stages AS (
    SELECT
        status.id AS status_id,
        status.ordering,
        current_stage.current_ordering,
        ROW_NUMBER() OVER (
            ORDER BY status.ordering
        ) AS stage_number
    FROM prm_ai.status status
    CROSS JOIN current_stage
    WHERE status.disabled = false
)
INSERT INTO prm_ai.agent_status_sla (
    ai_agent_id,
    agent_status_id,
    planned_date,
    completed_date
)
SELECT
    546,
    stage.status_id,
    CASE
        -- Прошедшие этапы специально оставляем без срока
        WHEN stage.ordering < stage.current_ordering
            THEN NULL

        -- Текущий и будущие этапы полностью заполняем
        ELSE (
            CURRENT_DATE
            + (10 + stage.stage_number::integer * 10)
        )::timestamp
    END,
    NULL
FROM stages stage
RETURNING
    ai_agent_id,
    agent_status_id,
    planned_date,
    completed_date;

Проверь созданный сценарий:

SELECT
    a.id,
    a.agent_id,
    current_status.code AS current_status,
    current_status.ordering AS current_ordering,
    stage.code AS stage,
    stage.ordering AS stage_ordering,
    sla.planned_date,
    sla.completed_date,
    CASE
        WHEN stage.ordering < current_status.ordering THEN 'PAST'
        WHEN stage.ordering = current_status.ordering THEN 'CURRENT'
        ELSE 'FUTURE'
    END AS stage_type
FROM prm_ai.ai_agent a
JOIN prm_ai.status current_status
    ON current_status.id = a.agent_status_id
JOIN prm_ai.status stage
    ON stage.disabled = false
LEFT JOIN prm_ai.agent_status_sla sla
    ON sla.ai_agent_id = a.id
   AND sla.agent_status_id = stage.id
WHERE a.id = 546
ORDER BY stage.ordering;

  ```
