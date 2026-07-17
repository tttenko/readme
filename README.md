```java
2. Добавление тестовых значений

Сейчас июль 2026 года, поэтому ручка читает:

предыдущий период — 2026-06-01;
позапрошлый период — 2026-05-01.

Скрипт создаст значения для обоих режимов и всех четырёх метрик:

BEGIN;

WITH test_values AS (
    SELECT
        imt.id AS initiative_metric_type_id,
        md.id AS metric_directory_id,
        md.name AS metric_name,
        md.direction,
        test_period.period_month,
        CASE md.name
            /*
             * Улучшение на обычных положительных числах.
             */
            WHEN 'Точность' THEN
                CASE
                    WHEN test_period.period_month = DATE '2026-05-01'
                        THEN 100
                    WHEN md.direction = 'more_is_better'
                        THEN 120
                    ELSE 80
                END

            /*
             * Ухудшение на обычных положительных числах.
             */
            WHEN 'CSI' THEN
                CASE
                    WHEN test_period.period_month = DATE '2026-05-01'
                        THEN 100
                    WHEN md.direction = 'more_is_better'
                        THEN 80
                    ELSE 120
                END

            /*
             * Улучшение на отрицательных числах.
             */
            WHEN 'Охват' THEN
                CASE
                    WHEN md.direction = 'more_is_better'
                         AND test_period.period_month = DATE '2026-05-01'
                        THEN -3
                    WHEN md.direction = 'more_is_better'
                        THEN -2
                    WHEN test_period.period_month = DATE '2026-05-01'
                        THEN -2
                    ELSE -3
                END

            /*
             * Ухудшение на отрицательных числах.
             */
            WHEN 'Скорость' THEN
                CASE
                    WHEN md.direction = 'more_is_better'
                         AND test_period.period_month = DATE '2026-05-01'
                        THEN -2
                    WHEN md.direction = 'more_is_better'
                        THEN -3
                    WHEN test_period.period_month = DATE '2026-05-01'
                        THEN -3
                    ELSE -2
                END
        END::numeric AS metric_value,
        150::numeric AS target_value
    FROM initiative_metric_type imt
    CROSS JOIN metrics_directory md
    CROSS JOIN (
        VALUES
            (DATE '2026-05-01'),
            (DATE '2026-06-01')
    ) AS test_period(period_month)
    WHERE imt.ai_agent_id = 109
      AND imt.agent_type IN ('autonomous', 'copilot')
      AND md.name IN (
          'Точность',
          'CSI',
          'Охват',
          'Скорость'
      )
),
numbered_values AS (
    SELECT
        (
            SELECT COALESCE(MAX(id), 0)
            FROM initiative_metric_value
        ) + ROW_NUMBER() OVER (
            ORDER BY
                initiative_metric_type_id,
                metric_directory_id,
                period_month
        ) AS id,
        initiative_metric_type_id,
        metric_directory_id,
        period_month,
        metric_value,
        target_value
    FROM test_values
)
INSERT INTO initiative_metric_value (
    id,
    initiative_agent_type_id,
    metric_directory_id,
    period_month,
    metric_value,
    target_value
)
SELECT
    id,
    initiative_metric_type_id,
    metric_directory_id,
    period_month,
    metric_value,
    target_value
FROM numbered_values;

COMMIT;

Должно добавиться:

2 режима × 4 метрики × 2 периода = 16 строк
3. Проверка добавленных данных
SELECT
    imv.id,
    imt.ai_agent_id,
    imt.agent_type,
    md.name AS metric_name,
    md.direction,
    imv.period_month,
    imv.metric_value,
    imv.target_value
FROM initiative_metric_value imv
JOIN initiative_metric_type imt
    ON imt.id = imv.initiative_agent_type_id
JOIN metrics_directory md
    ON md.id = imv.metric_directory_id
WHERE imt.ai_agent_id = 109
  AND imt.agent_type IN ('autonomous', 'copilot')
  AND md.name IN (
      'Точность',
      'CSI',
      'Охват',
      'Скорость'
  )
  AND imv.period_month IN (
      DATE '2026-05-01',
      DATE '2026-06-01'
  )
ORDER BY
    imt.agent_type,
    md.name,
    imv.period_month;

Должно вернуться 16 строк.
```
