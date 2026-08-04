```java
SELECT
    current_status.code AS current_status,
    current_status.ordering AS current_ordering,
    stage.id AS stage_id,
    stage.code AS stage,
    stage.ordering AS stage_ordering,
    sla.planned_date,
    sla.completed_date,
    CASE
        WHEN sla.ai_agent_id IS NULL THEN 'SLA_ROW_MISSING'
        WHEN sla.planned_date IS NULL THEN 'PLANNED_DATE_MISSING'
        ELSE 'FILLED'
    END AS result
FROM prm_ai.ai_agent agent
JOIN prm_ai.status current_status
    ON current_status.id = agent.agent_status_id
JOIN prm_ai.status stage
    ON stage.disabled = false
   AND stage.ordering >= current_status.ordering
LEFT JOIN prm_ai.agent_status_sla sla
    ON sla.ai_agent_id = agent.id
   AND sla.agent_status_id = stage.id
WHERE agent.id = 733
ORDER BY stage.ordering;
  ```
