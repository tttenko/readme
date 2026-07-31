```java
WITH expected_indexes(index_name) AS (
    VALUES
        ('idx_agent_contact_user_id_agent_id'),
        ('idx_agent_status_sla_agent_completed_planned'),
        ('idx_jira_issue_agent_project_key'),
        ('idx_initiative_metric_type_agent_id_type')
)
SELECT
    expected.index_name,
    indexes.tablename,
    indexes.indexdef,
    CASE
        WHEN indexes.indexname IS NOT NULL THEN 'OK'
        ELSE 'MISSING'
    END AS status
FROM expected_indexes expected
LEFT JOIN pg_indexes indexes
    ON indexes.indexname = expected.index_name
    AND indexes.schemaname = 'prm_ai'
ORDER BY expected.index_name;
  ```
