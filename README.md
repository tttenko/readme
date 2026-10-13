```java


SELECT id, agent_name, disabled
FROM ai_agent
WHERE disabled = false
ORDER BY id DESC
LIMIT 10;
```
