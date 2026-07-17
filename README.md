```java
BEGIN;

INSERT INTO metrics_directory (
    id,
    name,
    unit,
    direction,
    description,
    frequency,
    copilot_applicability,
    autonomous_applicability,
    requires_appeals_work,
    is_active,
    updated_by,
    updated_at
)
SELECT
    test_metric.id::uuid,
    test_metric.name,
    test_metric.unit,
    test_metric.direction,
    test_metric.description,
    test_metric.frequency,
    test_metric.copilot_applicability,
    test_metric.autonomous_applicability,
    test_metric.requires_appeals_work,
    test_metric.is_active,
    COALESCE(
        (
            SELECT md.updated_by
            FROM metrics_directory md
            WHERE md.updated_by IS NOT NULL
            LIMIT 1
        ),
        1
    ),
    CURRENT_TIMESTAMP
FROM (
    VALUES
        (
            '10000000-0000-0000-0000-000000000001',
            'Точность',
            'percent',
            'more_is_better',
            'Точность работы агента',
            'regular',
            TRUE,
            TRUE,
            FALSE,
            TRUE
        ),
        (
            '10000000-0000-0000-0000-000000000002',
            'CSI',
            'percent',
            'more_is_better',
            'Индекс удовлетворенности CSI',
            'regular',
            TRUE,
            TRUE,
            FALSE,
            TRUE
        ),
        (
            '10000000-0000-0000-0000-000000000003',
            'Охват',
            'percent',
            'more_is_better',
            'Охват процессов',
            'regular',
            TRUE,
            TRUE,
            FALSE,
            TRUE
        ),
        (
            '10000000-0000-0000-0000-000000000004',
            'Скорость',
            'percent',
            'less_is_better',
            'Скорость выполнения операций',
            'regular',
            TRUE,
            TRUE,
            FALSE,
            TRUE
        )
) AS test_metric(
    id,
    name,
    unit,
    direction,
    description,
    frequency,
    copilot_applicability,
    autonomous_applicability,
    requires_appeals_work,
    is_active
)
WHERE NOT EXISTS (
    SELECT 1
    FROM metrics_directory existing_metric
    WHERE existing_metric.name = test_metric.name
);

COMMIT;

Скрипт:

добавляет только отсутствующие метрики;
делает их доступными для autonomous и copilot;
устанавливает is_active = true;
не требует функции gen_random_uuid(), поскольку UUID заданы явно;
при повторном запуске не создаёт дубликаты по названию.

Проверь результат:

SELECT
    id,
    name,
    unit,
    direction,
    description,
    frequency,
    is_active,
    autonomous_applicability,
    copilot_applicability,
    requires_appeals_work,
    updated_by,
    updated_at
FROM metrics_directory
WHERE name IN (
    'Точность',
    'CSI',
    'Охват',
    'Скорость'
)
ORDER BY name;
```
