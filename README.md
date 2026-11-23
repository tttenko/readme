```java

SELECT
    s.id AS status_id,
    s.code,
    s.ordering,
    sla.planned_date,
    sla.completed_date,
    CASE
        WHEN sla.ai_agent_id IS NULL
            THEN 'Нет записи SLA'
        WHEN sla.completed_date IS NULL
             AND sla.planned_date IS NULL
            THEN 'Нет дедлайна'
        ELSE 'Условие заполнено'
    END AS deviation_reason
FROM ai_agent a
JOIN status current_status
    ON current_status.id = a.agent_status_id
JOIN status s
    ON s.disabled = false
    AND s.ordering >= current_status.ordering
LEFT JOIN agent_status_sla sla
    ON sla.agent_status_id = s.id
    AND sla.ai_agent_id = a.id
WHERE a.id = 12345
ORDER BY s.ordering;


```
