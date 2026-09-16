```java
SELECT
    imt.id AS initiative_metric_type_id,
    imt.ai_agent_id AS initiative_id,
    aa.agent_name,
    imt.agent_type,
    md.id AS metric_id,
    md.name AS metric_name,
    md.frequency
FROM initiative_metric_type imt
JOIN ai_agent aa
    ON aa.id = imt.ai_agent_id
CROSS JOIN metrics_directory md
WHERE md.is_active = true
  AND md.frequency = 'regular'
  AND NOT EXISTS (
      SELECT 1
      FROM initiative_metric_assignment ima
      WHERE ima.initiative_agent_type_id = imt.id
        AND ima.metric_id = md.id
  )
LIMIT 10;
```
