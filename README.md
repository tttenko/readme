```java

SELECT COUNT(DISTINCT a.id)
FROM ai_agent a
JOIN status current_status
    ON current_status.id = a.agent_status_id
JOIN status future_status
    ON future_status.disabled = false
    AND future_status.ordering >= current_status.ordering
LEFT JOIN agent_status_sla sla
    ON sla.ai_agent_id = a.id
    AND sla.agent_status_id = future_status.id
WHERE a.disabled = false
  AND sla.ai_agent_id IS NULL;
```
