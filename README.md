```java
UPDATE prm_ai.agent_status_sla sla
SET planned_date = NULL
FROM prm_ai.ai_agent agent
WHERE agent.id = 546
  AND sla.ai_agent_id = agent.id
  AND sla.agent_status_id = agent.agent_status_id;
  ```
