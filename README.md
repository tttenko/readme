```java
SELECT
    a.id,
    current_status.id AS current_status_id,
    current_status.code AS current_status,
    current_status.ordering AS current_ordering,
    stage.id AS stage_id,
    stage.code AS stage_code,
    stage.ordering AS stage_ordering,
    sla.planned_date,
    sla.completed_date
FROM prm_ai.ai_agent a
JOIN prm_ai.status current_status
    ON current_status.id = a.agent_status_id
JOIN prm_ai.status stage
    ON stage.disabled = false
LEFT JOIN prm_ai.agent_status_sla sla
    ON sla.ai_agent_id = a.id
   AND sla.agent_status_id = stage.id
WHERE a.id = 581
ORDER BY stage.ordering;
  ```
