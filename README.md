```java

Выполни два запроса:

SELECT
    column_name,
    column_default
FROM information_schema.columns
WHERE table_schema = 'prm_ai'
  AND table_name = 'jira_issue'
  AND column_name = 'id';

и:

SELECT
    id,
    agent_id,
    type,
    project,
    jira_key
FROM prm_ai.jira_issue
WHERE lower(project) = 'gigausage'
LIMIT 10;

```
