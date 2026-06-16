```java
SELECT
    aa.id,
    aa.agent_name,
    aa.import_status,
    aa.updated_by,
    aa.jira_from_status,
    aa.jira_from_updated,
    aa.jira_error_count
FROM ai_agent aa
WHERE aa.id = <agent_id>;

Потом посмотри его вехи:

SELECT
    aqg.ai_agent_id,
    aqg.quality_gate_code,
    aqg.state,
    aqg.updated
FROM agent_quality_gate aqg
WHERE aqg.ai_agent_id = <agent_id>;
```
