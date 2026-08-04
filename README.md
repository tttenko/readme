```java
SELECT
    a.id,
    s.id AS status_id,
    s.code,
    s.ordering
FROM prm_ai.ai_agent a
JOIN prm_ai.status s ON s.id = a.agent_status_id
WHERE a.id = 581;

BEGIN;

-- Предыдущий этап:
-- intentionally planned_date = NULL,
-- чтобы проверить, что прошедший этап не участвует в отклонениях.
WITH current_stage AS (
    SELECT current_status.ordering
    FROM prm_ai.ai_agent agent
    JOIN prm_ai.status current_status
        ON current_status.id = agent.agent_status_id
    WHERE agent.id = 581
),
previous_stage AS (
    SELECT status.id
    FROM prm_ai.status status
    CROSS JOIN current_stage
    WHERE status.disabled = false
      AND status.ordering < current_stage.ordering
    ORDER BY status.ordering DESC
    LIMIT 1
)
INSERT INTO prm_ai.agent_status_sla (
    ai_agent_id,
    agent_status_id,
    planned_date,
    completed_date
)
SELECT
    581,
    previous_stage.id,
    NULL,
    NULL
FROM previous_stage
ON CONFLICT (ai_agent_id, agent_status_id)
DO UPDATE SET
    planned_date = NULL,
    completed_date = NULL;


-- Текущий и все будущие этапы:
-- ставим даты дальше чем через 3 дня,
-- чтобы не получить другие deadline-отклонения.
WITH current_stage AS (
    SELECT current_status.ordering
    FROM prm_ai.ai_agent agent
    JOIN prm_ai.status current_status
        ON current_status.id = agent.agent_status_id
    WHERE agent.id = 581
),
current_and_future_stages AS (
    SELECT
        status.id,
        ROW_NUMBER() OVER (ORDER BY status.ordering) AS stage_number
    FROM prm_ai.status status
    CROSS JOIN current_stage
    WHERE status.disabled = false
      AND status.ordering >= current_stage.ordering
)
INSERT INTO prm_ai.agent_status_sla (
    ai_agent_id,
    agent_status_id,
    planned_date,
    completed_date
)
SELECT
    581,
    stage.id,
    (
        CURRENT_DATE
        + (10 + stage.stage_number::integer * 10)
    )::timestamp,
    NULL
FROM current_and_future_stages stage
ON CONFLICT (ai_agent_id, agent_status_id)
DO UPDATE SET
    planned_date = EXCLUDED.planned_date,
    completed_date = NULL;

COMMIT;

SELECT
    a.id,
    current_status.id AS current_status_id,
    current_status.code AS current_status,
    current_status.ordering AS current_ordering,
    stage.id AS stage_id,
    stage.code AS stage_code,
    stage.ordering AS stage_ordering,
    sla.planned_date,
    sla.completed_date
FROM prm_ai.ai_agent a
JOIN prm_ai.status current_status
    ON current_status.id = a.agent_status_id
JOIN prm_ai.status stage
    ON stage.disabled = false
LEFT JOIN prm_ai.agent_status_sla sla
    ON sla.ai_agent_id = a.id
   AND sla.agent_status_id = stage.id
WHERE a.id = 581
ORDER BY stage.ordering;
  ```
