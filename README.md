```java

BEGIN;

-- ============================================================
-- 1. Удаление старых тестовых данных для повторного запуска
-- ============================================================

DELETE FROM prm_ai.initiative_metric_value
WHERE initiative_agent_type_id IN (
    SELECT imt.id
    FROM prm_ai.initiative_metric_type imt
    JOIN prm_ai.ai_agent a
        ON a.id = imt.ai_agent_id
    WHERE a.agent_id LIKE 'SHOWCASE-DEV-%'
);

DELETE FROM prm_ai.initiative_metric_type
WHERE ai_agent_id IN (
    SELECT id
    FROM prm_ai.ai_agent
    WHERE agent_id LIKE 'SHOWCASE-DEV-%'
);

DELETE FROM prm_ai.agent_status_sla
WHERE ai_agent_id IN (
    SELECT id
    FROM prm_ai.ai_agent
    WHERE agent_id LIKE 'SHOWCASE-DEV-%'
);

DELETE FROM prm_ai.jira_issue
WHERE agent_id IN (
    SELECT id
    FROM prm_ai.ai_agent
    WHERE agent_id LIKE 'SHOWCASE-DEV-%'
);

DELETE FROM prm_ai.agent_enabler
WHERE agent_id IN (
    SELECT id
    FROM prm_ai.ai_agent
    WHERE agent_id LIKE 'SHOWCASE-DEV-%'
);

DELETE FROM prm_ai.ai_agent
WHERE agent_id LIKE 'SHOWCASE-DEV-%';


-- ============================================================
-- 2. Создание инициатив
-- ============================================================

INSERT INTO prm_ai.ai_agent (
    agent_id,
    agent_name,
    agent_description,
    agent_status_id,
    disabled,
    created
)
SELECT
    source.agent_id,
    source.agent_name,
    'Тестовая инициатива для проверки отклонений showcase',
    status.id,
    false,
    CURRENT_TIMESTAMP
FROM (
    VALUES
        (
            'SHOWCASE-DEV-NO-DEVIATIONS',
            'DEV No deviations',
            'research'
        ),
        (
            'SHOWCASE-DEV-DEADLINE-EXPIRED',
            'DEV Deadline expired',
            'research'
        ),
        (
            'SHOWCASE-DEV-METRICS-MISSING',
            'DEV Metrics missing',
            'targetSolution'
        ),
        (
            'SHOWCASE-DEV-METRICS-COMPLETE',
            'DEV Metrics complete',
            'targetSolution'
        ),
        (
            'SHOWCASE-DEV-DEADLINES-MISSING',
            'DEV Deadlines missing',
            'research'
        ),
        (
            'SHOWCASE-DEV-GIGAUSAGE-MISSING',
            'DEV GigaUsage missing',
            'research'
        ),
        (
            'SHOWCASE-DEV-ENABLERS-MISSING',
            'DEV Enablers missing',
            'research'
        ),
        (
            'SHOWCASE-DEV-DEADLINE-TOMORROW',
            'DEV Deadline tomorrow',
            'research'
        ),
        (
            'SHOWCASE-DEV-DEADLINE-IN-2-DAYS',
            'DEV Deadline in 2 days',
            'research'
        ),
        (
            'SHOWCASE-DEV-DEADLINE-IN-3-DAYS',
            'DEV Deadline in 3 days',
            'research'
        ),
        (
            'SHOWCASE-DEV-MULTIPLE-DEVIATIONS',
            'DEV Multiple deviations',
            'research'
        ),
        (
            'SHOWCASE-DEV-COMPLETED-STAGE',
            'DEV Completed stage',
            'research'
        ),

        -- Данные для отдельной проверки пагинации
        (
            'SHOWCASE-DEV-PAGE-A',
            'A-CLEAN',
            'research'
        ),
        (
            'SHOWCASE-DEV-PAGE-B',
            'B-DEVIATION',
            'research'
        ),
        (
            'SHOWCASE-DEV-PAGE-C',
            'C-CLEAN',
            'research'
        ),
        (
            'SHOWCASE-DEV-PAGE-D',
            'D-DEVIATION',
            'research'
        ),
        (
            'SHOWCASE-DEV-PAGE-E',
            'E-DEVIATION',
            'research'
        )
) AS source(
    agent_id,
    agent_name,
    status_code
)
JOIN prm_ai.status status
    ON status.code = source.status_code;


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


COMMIT;

```
