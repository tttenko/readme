```java

SELECT
    pid,
    state,
    wait_event_type,
    wait_event,
    NOW() - query_start AS query_duration,
    NOW() - xact_start AS transaction_duration,
    pg_blocking_pids(pid) AS blocked_by,
    LEFT(query, 300) AS current_query
FROM pg_stat_activity
WHERE datname = current_database()
  AND pid <> pg_backend_pid()
  AND state <> 'idle'
ORDER BY query_start;

```
