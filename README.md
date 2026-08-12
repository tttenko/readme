```java

BEGIN;

-- Посмотреть состояние до изменения
SELECT
    COUNT(*) FILTER (WHERE disabled = FALSE) AS active_count,
    COUNT(*) FILTER (WHERE disabled = TRUE)  AS disabled_count,
    COUNT(*) FILTER (WHERE disabled IS NULL) AS null_disabled_count
FROM ai_agent;


-- Активируем 50 отключённых агентов
WITH agents_to_activate AS (
    SELECT id
    FROM ai_agent
    WHERE disabled = TRUE
    ORDER BY id
    LIMIT 50
)
UPDATE ai_agent a
SET disabled = FALSE
FROM agents_to_activate ata
WHERE a.id = ata.id;


-- Проверяем результат
SELECT
    COUNT(*) FILTER (WHERE disabled = FALSE) AS active_count,
    COUNT(*) FILTER (WHERE disabled = TRUE)  AS disabled_count
FROM ai_agent;

COMMIT;

```
