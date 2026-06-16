```java
SELECT
    ji.id,
    ji.agent_id,
    ji.quality_gate_code,
    ji.jira_key
FROM jira_issue ji
JOIN quality_gate qg
    ON ji.quality_gate_code = qg.code
WHERE qg.type = 'quality_gate'
  AND qg.code = '<qualityGateCode>'
  AND ji.agent_id = <agent_id>
  AND ji.jira_key IS NOT NULL;
```
