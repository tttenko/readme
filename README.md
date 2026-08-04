```java
SELECT
    a.id,
    current_status.code AS current_status,
    current_status.ordering AS current_ordering,
    stage.id AS stage_id,
    stage.code AS stage_code,
    stage.ordering AS stage_ordering,
    sla.planned_date,
    sla.completed_date
FROM ai_agent a
JOIN status current_status
    ON current_status.id = a.agent_status_id
JOIN status stage
    ON stage.ordering >= current_status.ordering
   AND stage.disabled = false
LEFT JOIN agent_status_sla sla
    ON sla.ai_agent_id = a.id
   AND sla.agent_status_id = stage.id
WHERE a.id = 573
ORDER BY stage.ordering;
  ```
