```java

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

```
