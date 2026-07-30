```java
SELECT column_name
FROM information_schema.columns
WHERE table_schema = 'prm_ai'
  AND table_name = 'agent_status_sla'
  AND column_name IN (
      'ai_agent_id',
      'completed_date',
      'planned_date'
  );
  ```
