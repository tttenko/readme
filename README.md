```java
SELECT
    a.id,
    a.agent_id,
    a.agent_name,
    current_status.code AS current_status,
    current_status.ordering AS current_ordering,
    COUNT(sla.ai_agent_id) AS sla_count
FROM prm_ai.ai_agent a
JOIN prm_ai.status current_status
    ON current_status.id = a.agent_status_id
LEFT JOIN prm_ai.agent_status_sla sla
    ON sla.ai_agent_id = a.id
WHERE a.disabled = false
  AND current_status.ordering > (
      SELECT MIN(s.ordering)
      FROM prm_ai.status s
      WHERE s.disabled = false
  )
GROUP BY
    a.id,
    a.agent_id,
    a.agent_name,
    current_status.code,
    current_status.ordering
HAVING COUNT(sla.ai_agent_id) = 0
ORDER BY a.id
LIMIT 20;
  ```
