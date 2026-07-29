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

```
