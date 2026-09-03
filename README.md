```java
1. Общее количество инициатив + active/archive
SELECT
    COUNT(*) AS total,
    COUNT(*) FILTER (WHERE disabled = false) AS active,
    COUNT(*) FILTER (WHERE disabled = true) AS archived
FROM ai_agent;

Это позволит проверить:

число строк данных в Excel = total

и колонку:

disabled = false → Архив = Нет
disabled = true  → Архив = Да
2. Все статусы и их ordering
SELECT
    id,
    code,
    name,
    ordering,
    disabled
FROM status
ORDER BY ordering, id;

Мне нужен весь результат.

Особенно интересуют:

research
analysis
development
pilot
release
targetSolution
3. Найти по одной инициативе каждого интересующего статуса
SELECT DISTINCT ON (s.code)
    a.id,
    a.agent_id,
    a.agent_name,
    a.disabled,
    s.code AS current_status,
    s.name AS current_status_name,
    s.ordering
FROM ai_agent a
JOIN status s
    ON s.id = a.agent_status_id
WHERE s.code IN (
    'research',
    'analysis',
    'development',
    'pilot',
    'release',
    'targetSolution'
)
ORDER BY s.code, a.id;

Это даст нам кандидатов для проверки lifecycle.

Например получится что-то вроде:

analysis        id=10
development     id=30
pilot           id=55
research        id=71
release         id=90
targetSolution  id=110

Пришли весь результат.

4. Несколько архивных и активных инициатив

Архивные:

SELECT
    a.id,
    a.agent_id,
    a.agent_name,
    a.disabled,
    s.code AS current_status
FROM ai_agent a
LEFT JOIN status s
    ON s.id = a.agent_status_id
WHERE a.disabled = true
ORDER BY a.id
LIMIT 10;

Активные:

SELECT
    a.id,
    a.agent_id,
    a.agent_name,
    a.disabled,
    s.code AS current_status
FROM ai_agent a
LEFT JOIN status s
    ON s.id = a.agent_status_id
WHERE a.disabled = false
ORDER BY a.id
LIMIT 10;


9. Для проверки email — сначала один технический запрос

Поскольку в AgentContactEntity нет явного @JoinColumn для contact, я не хочу угадывать физическое имя FK.

Выполни:

SELECT
    table_name,
    column_name,
    data_type
FROM information_schema.columns
WHERE table_name IN (
    'agent_contact',
    'contact'
)
ORDER BY table_name, ordinal_position;

Пришли результат.

После этого я дам тебе точный SQL, который для каждой инициативы сформирует ожидаемую строку:

email1;email2;email3

ровно так же, как новый backend.

10. Дополнительно — кандидаты с максимальным количеством данных

Это полезно, чтобы в Excel не проверять полупустую строку:

SELECT
    a.id,
    a.agent_id,
    a.agent_name,
    a.disabled,
    s.code AS current_status,
    COUNT(DISTINCT ass.agent_status_id) AS sla_count,
    COUNT(DISTINCT ac.id) AS contact_count
FROM ai_agent a
LEFT JOIN status s
    ON s.id = a.agent_status_id
LEFT JOIN agent_status_sla ass
    ON ass.ai_agent_id = a.id
LEFT JOIN agent_contact ac
    ON ac.agent_id = a.id
GROUP BY
    a.id,
    a.agent_id,
    a.agent_name,
    a.disabled,
    s.code
ORDER BY
    sla_count DESC,
    contact_count DESC,
    a.id
LIMIT 20;
```
