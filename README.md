```java
UPDATE agent_status_sla sla
SET planned_date = '2026-08-20 00:00:00'
FROM status s
WHERE sla.agent_status_id = s.id
  AND sla.ai_agent_id = 545
  AND s.code = 'pilot';
  ```
