```java

1. Убедимся, что у 546 нет валидного GigaUsage

Сначала посмотри строки:

SELECT *
FROM prm_ai.jira_issue
WHERE ai_agent_id = 546;

Если колонка FK называется не ai_agent_id, выполни:

SELECT column_name
FROM information_schema.columns
WHERE table_schema = 'prm_ai'
  AND table_name = 'jira_issue'
ORDER BY ordinal_position;

Если ai_agent_id правильный, отдельно проверь GigaUsage:

SELECT *
FROM prm_ai.jira_issue
WHERE ai_agent_id = 546
  AND lower(project) = 'gigausage';

Для чистого теста можно удалить GigaUsage этого тестового агента:

DELETE FROM prm_ai.jira_issue
WHERE ai_agent_id = 546
  AND lower(project) = 'gigausage';

Теперь:

SELECT *
FROM prm_ai.jira_issue
WHERE ai_agent_id = 546
  AND lower(project) = 'gigausage';

должен вернуть 0 rows.

2. Проверяем research
UPDATE prm_ai.ai_agent
SET agent_status_id = (
    SELECT id
    FROM prm_ai.status
    WHERE code = 'research'
)
WHERE id = 546;

Проверка:

SELECT
    a.id,
    a.agent_id,
    s.code,
    s.name
FROM prm_ai.ai_agent a
JOIN prm_ai.status s ON s.id = a.agent_status_id
WHERE a.id = 546;

Должно быть:

546 | PULT546 | research | Гипотеза

В Swagger вызываешь тот же:

POST /api/v1/ai-agent/showcase

с поиском PULT546.

По документации не должно быть:

{
  "code": "GIGAUSAGE_NOT_FILLED"
}

Но по текущему коду я ожидаю, что оно появится. Если появится — первый баг уже подтверждён.

3. Проверяем analysis
UPDATE prm_ai.ai_agent
SET agent_status_id = (
    SELECT id
    FROM prm_ai.status
    WHERE code = 'analysis'
)
WHERE id = 546;

Swagger повторяем.

Ожидание по документации:

GIGAUSAGE_NOT_FILLED — НЕТ

По текущей реализации, скорее всего:

GIGAUSAGE_NOT_FILLED — ЕСТЬ

Это тоже ошибка.

4. Проверяем development
UPDATE prm_ai.ai_agent
SET agent_status_id = (
    SELECT id
    FROM prm_ai.status
    WHERE code = 'development'
)
WHERE id = 546;

Swagger повторяем.

По документации:

GIGAUSAGE_NOT_FILLED — НЕТ

Если приходит — ошибка.

5. Проверяем MVP (pilot)
UPDATE prm_ai.ai_agent
SET agent_status_id = (
    SELECT id
    FROM prm_ai.status
    WHERE code = 'pilot'
)
WHERE id = 546;

Проверка:

SELECT
    a.id,
    s.code,
    s.name,
    s.ordering
FROM prm_ai.ai_agent a
JOIN prm_ai.status s ON s.id = a.agent_status_id
WHERE a.id = 546;

Получим:

pilot | MVP | 40

Теперь Swagger.

Здесь уже должно появиться:

{
  "code": "GIGAUSAGE_NOT_FILLED",
  "priority": 4,
  "title": "Добавить",
  "description": "Добавьте GigaUsage"
}

Если появилось — этот сценарий работает корректно.

6. Проверяем targetSolution
UPDATE prm_ai.ai_agent
SET agent_status_id = (
    SELECT id
    FROM prm_ai.status
    WHERE code = 'targetSolution'
)
WHERE id = 546;

При отсутствии GigaUsage Swagger также должен вернуть:

GIGAUSAGE_NOT_FILLED

Это корректное поведение.

На этом этапе уже можно будет доказать основной баг

С высокой вероятностью получим:

research        → GIGAUSAGE_NOT_FILLED ❌
analysis        → GIGAUSAGE_NOT_FILLED ❌
development     → GIGAUSAGE_NOT_FILLED ❌
pilot           → GIGAUSAGE_NOT_FILLED ✅
targetSolution  → GIGAUSAGE_NOT_FILLED ✅

То есть сама проверка отсутствия GigaUsage работает, но не применяется ограничение по статусу.

7. После этого проверим наличие корректного GigaUsage

Для этого я не хочу сейчас придумывать INSERT в jira_issue, потому что мы пока не видели обязательные поля этой таблицы.

После первых пяти проверок пришли результат:

SELECT column_name, data_type, is_nullable
FROM information_schema.columns
WHERE table_schema = 'prm_ai'
  AND table_name = 'jira_issue'
ORDER BY ordinal_position;

и я дам точный безопасный INSERT для агента 546, чтобы получить:

pilot + валидный GigaUsage → отклонения НЕТ
targetSolution + валидный GigaUsage → отклонения НЕТ

Это полностью закроет проверку требования.

```
