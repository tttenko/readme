```java

2. Проверка, что инициативы создались
SELECT
    agent.id,
    agent.agent_id,
    agent.agent_name,
    status.code AS status_code,
    agent.disabled,
    agent.created
FROM prm_ai.ai_agent agent
JOIN prm_ai.status status
    ON status.id = agent.agent_status_id
WHERE agent.agent_id LIKE 'SHOWCASE-DEV-%'
ORDER BY agent.agent_id;

Ожидается 17 инициатив.

3. Проверка SLA
SELECT
    agent.agent_id,
    sla.planned_date,
    sla.completed_date,
    status.code AS stage_status
FROM prm_ai.agent_status_sla sla
JOIN prm_ai.ai_agent agent
    ON agent.id = sla.ai_agent_id
JOIN prm_ai.status status
    ON status.id = sla.agent_status_id
WHERE agent.agent_id LIKE 'SHOWCASE-DEV-%'
ORDER BY agent.agent_id;

Особенно проверь:

SHOWCASE-DEV-DEADLINE-EXPIRED
planned_date = вчера
completed_date = NULL

SHOWCASE-DEV-DEADLINES-MISSING
planned_date = NULL
completed_date = NULL

SHOWCASE-DEV-COMPLETED-STAGE
planned_date = дата в прошлом
completed_date = заполнено
4. Проверка GigaUsage
SELECT
    agent.agent_id,
    issue.project,
    issue.jira_key
FROM prm_ai.ai_agent agent
LEFT JOIN prm_ai.jira_issue issue
    ON issue.agent_id = agent.id
   AND issue.project = 'gigausage'
WHERE agent.agent_id LIKE 'SHOWCASE-DEV-%'
ORDER BY agent.agent_id;

Без GigaUsage должны быть:

SHOWCASE-DEV-GIGAUSAGE-MISSING
SHOWCASE-DEV-MULTIPLE-DEVIATIONS
SHOWCASE-DEV-PAGE-E
5. Проверка энейблеров
SELECT
    agent.agent_id,
    enabler.id AS enabler_id,
    enabler.name AS enabler_name
FROM prm_ai.ai_agent agent
LEFT JOIN prm_ai.agent_enabler agent_enabler
    ON agent_enabler.agent_id = agent.id
LEFT JOIN prm_ai.enabler enabler
    ON enabler.id = agent_enabler.enabler_id
WHERE agent.agent_id LIKE 'SHOWCASE-DEV-%'
ORDER BY agent.agent_id;

Без энейблера должны быть:

SHOWCASE-DEV-ENABLERS-MISSING
SHOWCASE-DEV-MULTIPLE-DEVIATIONS
SHOWCASE-DEV-PAGE-D
6. Проверка метрик
SELECT
    agent.agent_id,
    metric_type.agent_type,
    metric.name,
    metric.code,
    value.period_month,
    value.metric_value,
    value.target_value
FROM prm_ai.ai_agent agent
JOIN prm_ai.initiative_metric_type metric_type
    ON metric_type.ai_agent_id = agent.id
LEFT JOIN prm_ai.initiative_metric_value value
    ON value.initiative_agent_type_id = metric_type.id
LEFT JOIN prm_ai.metrics_directory metric
    ON metric.id = value.metric_directory_id
WHERE agent.agent_id IN (
    'SHOWCASE-DEV-METRICS-MISSING',
    'SHOWCASE-DEV-METRICS-COMPLETE'
)
ORDER BY
    agent.agent_id,
    metric.name;

Ожидается:

у SHOWCASE-DEV-METRICS-COMPLETE все metric_value заполнены;
у SHOWCASE-DEV-METRICS-MISSING ровно один metric_value = NULL.

Проверить количество незаполненных значений:

SELECT
    agent.agent_id,
    COUNT(*) AS total_values,
    COUNT(value.metric_value) AS filled_values,
    COUNT(*) - COUNT(value.metric_value) AS missing_values
FROM prm_ai.ai_agent agent
JOIN prm_ai.initiative_metric_type metric_type
    ON metric_type.ai_agent_id = agent.id
JOIN prm_ai.initiative_metric_value value
    ON value.initiative_agent_type_id = metric_type.id
WHERE agent.agent_id IN (
    'SHOWCASE-DEV-METRICS-MISSING',
    'SHOWCASE-DEV-METRICS-COMPLETE'
)
GROUP BY
    agent.agent_id
ORDER BY
    agent.agent_id;
COMMIT;

-- ============================================================
-- 3. Создание SLA этапов
--
-- planned_day_offset:
--  -1 = вчера
--   1 = завтра
--   2 = через два дня
--   3 = через три дня
--  10 = безопасная дата без отклонения
-- NULL = дедлайн не заполнен
-- ============================================================

INSERT INTO prm_ai.agent_status_sla (
    ai_agent_id,
    agent_status_id,
    planned_date,
    completed_date
)
SELECT
    agent.id,
    agent.agent_status_id,

    CASE
        WHEN source.planned_day_offset IS NULL
            THEN NULL
        ELSE
            (CURRENT_DATE + source.planned_day_offset)::timestamp
    END,

    CASE
        WHEN source.completed_day_offset IS NULL
            THEN NULL
        ELSE
            (CURRENT_DATE + source.completed_day_offset)::timestamp
    END
FROM (
    VALUES
        ('SHOWCASE-DEV-NO-DEVIATIONS',          10, NULL),
        ('SHOWCASE-DEV-DEADLINE-EXPIRED',       -1, NULL),
        ('SHOWCASE-DEV-METRICS-MISSING',        10, NULL),
        ('SHOWCASE-DEV-METRICS-COMPLETE',       10, NULL),
        ('SHOWCASE-DEV-DEADLINES-MISSING',    NULL, NULL),
        ('SHOWCASE-DEV-GIGAUSAGE-MISSING',      10, NULL),
        ('SHOWCASE-DEV-ENABLERS-MISSING',       10, NULL),
        ('SHOWCASE-DEV-DEADLINE-TOMORROW',       1, NULL),
        ('SHOWCASE-DEV-DEADLINE-IN-2-DAYS',      2, NULL),
        ('SHOWCASE-DEV-DEADLINE-IN-3-DAYS',      3, NULL),
        ('SHOWCASE-DEV-MULTIPLE-DEVIATIONS',    -1, NULL),

        -- Просроченный, но уже завершённый этап:
        -- отклонений по срокам быть не должно.
        ('SHOWCASE-DEV-COMPLETED-STAGE',        -5,   -4),

        -- Пагинация
        ('SHOWCASE-DEV-PAGE-A',                 10, NULL),
        ('SHOWCASE-DEV-PAGE-B',                 -1, NULL),
        ('SHOWCASE-DEV-PAGE-C',                 10, NULL),
        ('SHOWCASE-DEV-PAGE-D',                 10, NULL),
        ('SHOWCASE-DEV-PAGE-E',                 10, NULL)
) AS source(
    agent_id,
    planned_day_offset,
    completed_day_offset
)
JOIN prm_ai.ai_agent agent
    ON agent.agent_id = source.agent_id;


-- ============================================================
-- 4. Добавление валидной GigaUsage-задачи
--
-- Не добавляем её для:
--  - SHOWCASE-DEV-GIGAUSAGE-MISSING
--  - SHOWCASE-DEV-MULTIPLE-DEVIATIONS
--  - SHOWCASE-DEV-PAGE-E
-- ============================================================

INSERT INTO prm_ai.jira_issue (
    agent_id,
    type,
    jira_key,
    project,
    created
)
SELECT
    agent.id,
    'initiative',
    'GIGA-TEST-' || agent.id,
    'gigausage',
    CURRENT_TIMESTAMP
FROM prm_ai.ai_agent agent
WHERE agent.agent_id LIKE 'SHOWCASE-DEV-%'
  AND agent.agent_id NOT IN (
      'SHOWCASE-DEV-GIGAUSAGE-MISSING',
      'SHOWCASE-DEV-MULTIPLE-DEVIATIONS',
      'SHOWCASE-DEV-PAGE-E'
  );


-- ============================================================
-- 5. Добавление энейблера
--
-- Используем существующий enabler_id = 1.
--
-- Не добавляем для:
--  - SHOWCASE-DEV-ENABLERS-MISSING
--  - SHOWCASE-DEV-MULTIPLE-DEVIATIONS
--  - SHOWCASE-DEV-PAGE-D
-- ============================================================

INSERT INTO prm_ai.agent_enabler (
    agent_id,
    enabler_id
)
SELECT
    agent.id,
    1
FROM prm_ai.ai_agent agent
WHERE agent.agent_id LIKE 'SHOWCASE-DEV-%'
  AND agent.agent_id NOT IN (
      'SHOWCASE-DEV-ENABLERS-MISSING',
      'SHOWCASE-DEV-MULTIPLE-DEVIATIONS',
      'SHOWCASE-DEV-PAGE-D'
  );


-- ============================================================
-- 6. Добавление режима autonomous для тестов метрик
-- ============================================================

INSERT INTO prm_ai.initiative_metric_type (
    ai_agent_id,
    agent_type
)
SELECT
    agent.id,
    'autonomous'
FROM prm_ai.ai_agent agent
WHERE agent.agent_id IN (
    'SHOWCASE-DEV-METRICS-MISSING',
    'SHOWCASE-DEV-METRICS-COMPLETE'
);


-- ============================================================
-- 7. Полностью заполненные метрики
--
-- Для SHOWCASE-DEV-METRICS-COMPLETE заполняем все активные
-- метрики, применимые к autonomous.
-- ============================================================

INSERT INTO prm_ai.initiative_metric_value (
    initiative_agent_type_id,
    metric_directory_id,
    period_month,
    metric_value,
    target_value
)
SELECT
    metric_type.id,
    metric.id,
    DATE_TRUNC('month', CURRENT_DATE)::date,
    100,
    100
FROM prm_ai.initiative_metric_type metric_type
JOIN prm_ai.ai_agent agent
    ON agent.id = metric_type.ai_agent_id
JOIN prm_ai.metrics_directory metric
    ON metric.is_active = true
   AND metric.autonomous_applicability = true
WHERE agent.agent_id = 'SHOWCASE-DEV-METRICS-COMPLETE'
  AND metric_type.agent_type = 'autonomous';


-- ============================================================
-- 8. Частично заполненные метрики
--
-- Для SHOWCASE-DEV-METRICS-MISSING создаём все строки,
-- но у одной метрики оставляем metric_value = NULL.
--
-- Это доказывает, что существование строки само по себе
-- не означает, что метрика заполнена.
-- ============================================================

WITH applicable_metrics AS (
    SELECT
        metric.id,
        ROW_NUMBER() OVER (ORDER BY metric.id) AS row_number
    FROM prm_ai.metrics_directory metric
    WHERE metric.is_active = true
      AND metric.autonomous_applicability = true
)
INSERT INTO prm_ai.initiative_metric_value (
    initiative_agent_type_id,
    metric_directory_id,
    period_month,
    metric_value,
    target_value
)
SELECT
    metric_type.id,
    metric.id,
    DATE_TRUNC('month', CURRENT_DATE)::date,

    CASE
        WHEN metric.row_number = 1
            THEN NULL
        ELSE
            100
    END,

    100
FROM prm_ai.initiative_metric_type metric_type
JOIN prm_ai.ai_agent agent
    ON agent.id = metric_type.ai_agent_id
CROSS JOIN applicable_metrics metric
WHERE agent.agent_id = 'SHOWCASE-DEV-METRICS-MISSING'
  AND metric_type.agent_type = 'autonomous';


```
