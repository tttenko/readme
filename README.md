```java
1. Перевести агента 546 на третий статус
UPDATE prm_ai.ai_agent
SET agent_status_id = (
    SELECT id
    FROM prm_ai.status
    WHERE code = 'development'
      AND disabled = false
    LIMIT 1
)
WHERE id = 546;

Проверить:

SELECT
    a.id,
    s.id AS status_id,
    s.code,
    s.ordering
FROM prm_ai.ai_agent a
JOIN prm_ai.status s
    ON s.id = a.agent_status_id
WHERE a.id = 546;

Ожидается:

development | ordering = 30
2. Удалить все старые SLA агента
DELETE FROM prm_ai.agent_status_sla
WHERE ai_agent_id = 546;

Теперь для этапов 1 и 2 строк действительно не будет.

3. Создать строки только для текущего и будущих этапов
WITH current_stage AS (
    SELECT current_status.ordering AS current_ordering
    FROM prm_ai.ai_agent agent
    JOIN prm_ai.status current_status
        ON current_status.id = agent.agent_status_id
    WHERE agent.id = 546
),
current_and_future_stages AS (
    SELECT
        status.id AS status_id,
        status.ordering,
        ROW_NUMBER() OVER (
            ORDER BY status.ordering
        ) AS stage_number
    FROM prm_ai.status status
    CROSS JOIN current_stage
    WHERE status.disabled = false
      AND status.ordering >= current_stage.current_ordering
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
    (
        CURRENT_DATE
        + (10 + stage.stage_number::integer * 10)
    )::timestamp,
    NULL
FROM current_and_future_stages stage
RETURNING
    ai_agent_id,
    agent_status_id,
    planned_date,
    completed_date;

В результате строки появятся для:

development
pilot
targetSolution
release

А для пройденных:

research
analysis

строк в agent_status_sla не будет вообще.

4. Проверить подготовленный сценарий
SELECT
    current_status.code AS current_status,
    current_status.ordering AS current_ordering,
    stage.code AS stage,
    stage.ordering AS stage_ordering,
    sla.ai_agent_id AS sla_exists,
    sla.planned_date,
    sla.completed_date,
    CASE
        WHEN stage.ordering < current_status.ordering THEN 'PAST'
        WHEN stage.ordering = current_status.ordering THEN 'CURRENT'
        ELSE 'FUTURE'
    END AS stage_type
FROM prm_ai.ai_agent agent
JOIN prm_ai.status current_status
    ON current_status.id = agent.agent_status_id
JOIN prm_ai.status stage
    ON stage.disabled = false
LEFT JOIN prm_ai.agent_status_sla sla
    ON sla.ai_agent_id = agent.id
   AND sla.agent_status_id = stage.id
WHERE agent.id = 546
ORDER BY stage.ordering;
  ```
