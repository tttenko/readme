```java
SELECT
    ai_agent_id,
    quality_gate_code,
    state,
    updated
FROM agent_quality_gate
WHERE ai_agent_id = <agent_id>
  AND quality_gate_code = '<qualityGateCode>';
```
