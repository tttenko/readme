```java
Как найти подходящий userId

Запусти в DBeaver:

SELECT
    user_id,
    COUNT(DISTINCT agent_id) AS initiatives_count
FROM (
    SELECT
        a.owner_id AS user_id,
        a.id AS agent_id
    FROM ai_agent a
    WHERE a.disabled = false
      AND a.owner_id IS NOT NULL

    UNION ALL

    SELECT
        ac.user_id,
        ac.agent_id
    FROM agent_contact ac
    JOIN ai_agent a ON a.id = ac.agent_id
    WHERE a.disabled = false
      AND ac.user_id IS NOT NULL
) users_with_initiatives
GROUP BY user_id
ORDER BY initiatives_count DESC;

Выбери пользователя, у которого, например, 2–10 инициатив, и передай его ID:

GET /api/v1/ai-agent/initiative/deviations/count?userId=12345
Сначала проверь количество инициатив пользователя
SELECT COUNT(DISTINCT a.id)
FROM ai_agent a
LEFT JOIN agent_contact ac ON ac.agent_id = a.id
WHERE a.disabled = false
  AND (
      a.owner_id = 12345
      OR ac.user_id = 12345
  );

Результат ручки не может быть больше этого числа.

Как проверить результат 44

Важно: 44 — это не количество всех найденных отклонений. Это количество уникальных инициатив, у каждой из которых найдено хотя бы одно отклонение.

Сначала проверь общее количество активных инициатив:

SELECT COUNT(*)
FROM ai_agent
WHERE disabled = false;

Если здесь тоже 44, скорее всего каждая инициатива попала хотя бы под одно массовое правило, например:

не заполнен GigaUsage;
не указаны энейблеры;
не заполнены сроки этапов.

Проверь самые вероятные правила отдельно.

Нет энейблеров
SELECT COUNT(*)
FROM ai_agent a
WHERE a.disabled = false
  AND NOT EXISTS (
      SELECT 1
      FROM agent_enabler ae
      WHERE ae.agent_id = a.id
  );
Нет GigaUsage
SELECT COUNT(*)
FROM ai_agent a
WHERE a.disabled = false
  AND NOT EXISTS (
      SELECT 1
      FROM jira_issue j
      WHERE j.agent_id = a.id
        AND LOWER(j.project) = 'gigausage'
        AND NULLIF(BTRIM(j.jira_key), '') IS NOT NULL
  );
Не заполнены сроки незавершённых этапов
SELECT COUNT(DISTINCT a.id)
FROM ai_agent a
JOIN agent_status_sla sla ON sla.ai_agent_id = a.id
WHERE a.disabled = false
  AND sla.completed_date IS NULL
  AND sla.planned_date IS NULL;
Есть просроченный этап
SELECT COUNT(DISTINCT a.id)
FROM ai_agent a
JOIN agent_status_sla sla ON sla.ai_agent_id = a.id
WHERE a.disabled = false
  AND sla.completed_date IS NULL
  AND sla.planned_date < CURRENT_DATE;

Поля planned_date и completed_date действительно используются для SLA этапов.

Самая точная проверка

Временно замени в native-запросе финальную часть:

SELECT COUNT(*)
FROM (
    SELECT id FROM cheap_deviations
    UNION ALL
    SELECT id FROM metric_deviations
) initiatives_with_deviations

на:

SELECT id
FROM cheap_deviations

UNION ALL

SELECT id
FROM metric_deviations

ORDER BY id;

И временно сделай метод репозитория возвращающим:

fun findInitiativeIdsWithDeviations(
    @Param("params")
    params: InitiativeDeviationCountQueryParams,
): List<Long>
  ```
