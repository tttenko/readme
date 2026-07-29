```java

SELECT
    c.table_name,
    c.ordinal_position,
    c.column_name,
    c.data_type,
    c.udt_name,
    c.is_nullable,
    c.column_default,
    c.is_identity,
    c.identity_generation
FROM information_schema.columns c
WHERE c.table_schema = 'public'
  AND c.table_name IN (
      'ai_agent',
      'agent_status_sla',
      'status',
      'jira_issue',
      'agent_enabler',
      'enabler',
      'initiative_metric_type',
      'initiative_metric_value',
      'metrics_directory',
      'program',
      'division',
      'block',
      'initiative_type',
      'user_audience'
  )
ORDER BY
    c.table_name,
    c.ordinal_position;

SELECT
    tc.table_name,
    tc.constraint_name,
    tc.constraint_type,
    kcu.column_name,
    ccu.table_name AS referenced_table,
    ccu.column_name AS referenced_column
FROM information_schema.table_constraints tc
LEFT JOIN information_schema.key_column_usage kcu
    ON kcu.constraint_schema = tc.constraint_schema
   AND kcu.constraint_name = tc.constraint_name
LEFT JOIN information_schema.constraint_column_usage ccu
    ON ccu.constraint_schema = tc.constraint_schema
   AND ccu.constraint_name = tc.constraint_name
WHERE tc.table_schema = 'public'
  AND tc.table_name IN (
      'ai_agent',
      'agent_status_sla',
      'status',
      'jira_issue',
      'agent_enabler',
      'enabler',
      'initiative_metric_type',
      'initiative_metric_value',
      'metrics_directory'
  )
ORDER BY
    tc.table_name,
    tc.constraint_type,
    tc.constraint_name,
    kcu.ordinal_position;

SELECT id, code, name, ordering, disabled
FROM status
ORDER BY ordering;
SELECT id, code, name
FROM program
ORDER BY id
LIMIT 10;
SELECT id, code, name
FROM block
ORDER BY id
LIMIT 10;
SELECT id, code, name, block_id
FROM division
ORDER BY id
LIMIT 10;
SELECT code
FROM initiative_type
ORDER BY code;
SELECT code
FROM user_audience
ORDER BY code;
SELECT id, name
FROM enabler
ORDER BY id
LIMIT 10;
SELECT
    id,
    code,
    name,
    is_active,
    autonomous_applicability,
    copilot_applicability,
    requires_appeals_work
FROM metrics_directory
WHERE is_active = true
ORDER BY name;

```
