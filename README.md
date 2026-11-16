```java
SELECT agent_id, COUNT(*) AS duplicate_count
FROM prm_ai.ai_agent
GROUP BY agent_id
HAVING COUNT(*) > 1
ORDER BY duplicate_count DESC;
```
