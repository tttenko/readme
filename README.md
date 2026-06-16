```java
SELECT
    ji.id,
    ji.agent_id,
    ji.quality_gate_code,
    ji.jira_key
FROM jira_issue ji
WHERE ji.agent_id = <agent_id>
  AND ji.quality_gate_code = '<qualityGateCode>'
  AND ji.jira_key IS NOT NULL;
```
