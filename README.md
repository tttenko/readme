```java
SELECT
    schemaname,
    tablename,
    indexname,
    indexdef
FROM pg_indexes
WHERE schemaname = current_schema()
  AND tablename IN (
      'agent_contact',
      'agent_status_sla',
      'jira_issue',
      'agent_enabler',
      'initiative_metric_type',
      'initiative_metric_value'
  )
ORDER BY
    tablename,
    indexname;
```
