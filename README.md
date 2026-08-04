```java
Создание тестовых данных для агента 581

Сначала убедись, что агент существует и посмотри его текущий статус:

SELECT
    a.id,
    s.id AS status_id,
    s.code,
    s.ordering
FROM prm_ai.ai_agent a
JOIN prm_ai.status s
    ON s.id = a.agent_status_id
WHERE a.id = 581;

Затем удали старые тестовые SLA-записи:

DELETE FROM prm_ai.agent_status_sla
WHERE ai_agent_id = 581;

Создай строки для всех этапов:

WITH current_stage AS (
    SELECT current_status.ordering AS current_ordering
    FROM prm_ai.ai_agent agent
    JOIN prm_ai.status current_status
        ON current_status.id = agent.agent_status_id
    WHERE agent.id = 581
),
stages AS (
    SELECT
        status.id AS status_id,
        status.ordering,
        current_stage.current_ordering,
        ROW_NUMBER() OVER (ORDER BY status.ordering) AS stage_number
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
    581,
    stage.status_id,
    CASE
        -- Для уже пройденных этапов специально не указываем срок
        WHEN stage.ordering < stage.current_ordering
            THEN NULL

        -- Для текущего и будущих этапов указываем сроки
        ELSE (
            CURRENT_DATE
            + (10 + stage.stage_number::integer * 10)
        )::timestamp
    END,
    NULL
FROM stages stage;
Что получится

Например, если агент находится на третьем этапе:

этап 1 — planned_date = null
этап 2 — planned_date = null
этап 3 — planned_date заполнена
этап 4 — planned_date заполнена
этап 5 — planned_date заполнена
этап 6 — planned_date заполнена

То есть прошедшие этапы специально останутся без сроков, а у текущего и будущих сроки будут заполнены.

Проверка данных
SELECT
    a.id,
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
WHERE a.id = 581
ORDER BY stage.ordering;
  ```
