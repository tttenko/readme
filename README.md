```java
SELECT
    a.id,
    current_status.code AS current_status,
    current_status.ordering AS current_ordering,
    stage_status.code AS stage_status,
    stage_status.ordering AS stage_ordering,
    sla.planned_date,
    sla.completed_date
FROM ai_agent a
JOIN status current_status
    ON current_status.id = a.agent_status_id
LEFT JOIN agent_status_sla sla
    ON sla.ai_agent_id = a.id
LEFT JOIN status stage_status
    ON stage_status.id = sla.agent_status_id
WHERE a.id = 545
ORDER BY stage_status.ordering;
  ```
