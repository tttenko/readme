```java

WITH ranked_agents AS (
    SELECT
        id,
        agent_id,
        agent_name,
        created,
        ROW_NUMBER() OVER (
            PARTITION BY agent_id
            ORDER BY id
        ) AS row_number
    FROM prm_ai.ai_agent
    WHERE agent_id LIKE 'SHOWCASE-DEV-%'
)
SELECT
    id,
    agent_id,
    agent_name,
    created
FROM ranked_agents
WHERE row_number > 1
ORDER BY id;

Должно вернуться 17 строк — вторые экземпляры инициатив.

2. Точечно удалить дубли

Запусти весь блок целиком:

BEGIN;

-- Сохраняем ID дублей во временную таблицу.
-- Для каждого agent_id остаётся запись с минимальным id.
CREATE TEMP TABLE duplicate_showcase_agent_ids
ON COMMIT DROP
AS
SELECT id
FROM (
    SELECT
        id,
        ROW_NUMBER() OVER (
            PARTITION BY agent_id
            ORDER BY id
        ) AS row_number
    FROM prm_ai.ai_agent
    WHERE agent_id LIKE 'SHOWCASE-DEV-%'
) ranked_agents
WHERE row_number > 1;


-- Удаляем значения метрик дублей
DELETE FROM prm_ai.initiative_metric_value
WHERE initiative_agent_type_id IN (
    SELECT metric_type.id
    FROM prm_ai.initiative_metric_type metric_type
    WHERE metric_type.ai_agent_id IN (
        SELECT id
        FROM duplicate_showcase_agent_ids
    )
);


-- Удаляем режимы работы дублей
DELETE FROM prm_ai.initiative_metric_type
WHERE ai_agent_id IN (
    SELECT id
    FROM duplicate_showcase_agent_ids
);


-- Удаляем SLA дублей
DELETE FROM prm_ai.agent_status_sla
WHERE ai_agent_id IN (
    SELECT id
    FROM duplicate_showcase_agent_ids
);


-- Удаляем Jira issue дублей
DELETE FROM prm_ai.jira_issue
WHERE agent_id IN (
    SELECT id
    FROM duplicate_showcase_agent_ids
);


-- Удаляем связи с энейблерами
DELETE FROM prm_ai.agent_enabler
WHERE agent_id IN (
    SELECT id
    FROM duplicate_showcase_agent_ids
);


-- Удаляем сами дубли инициатив
DELETE FROM prm_ai.ai_agent
WHERE id IN (
    SELECT id
    FROM duplicate_showcase_agent_ids
);

COMMIT;
3. Проверь результат
SELECT
    agent_id,
    COUNT(*) AS records_count,
    MIN(id) AS remaining_id
FROM prm_ai.ai_agent
WHERE agent_id LIKE 'SHOWCASE-DEV-%'
GROUP BY agent_id
ORDER BY agent_id;

Для каждой инициативы должно быть:

records_count = 1

Общее количество:

SELECT COUNT(*) AS agents_count
FROM prm_ai.ai_agent
WHERE agent_id LIKE 'SHOWCASE-DEV-%';

Ожидается:

agents_count = 17

Проверка, что дублей больше нет:

SELECT
    agent_id,
    COUNT(*) AS records_count
FROM prm_ai.ai_agent
WHERE agent_id LIKE 'SHOWCASE-DEV-%'
GROUP BY agent_id
HAVING COUNT(*) > 1;

```
