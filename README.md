```java

SELECT
    pid,
    datname,
    usename,
    application_name,
    client_addr,
    state,
    wait_event_type,
    wait_event,
    NOW() - xact_start AS transaction_duration,
    LEFT(query, 200) AS current_query
FROM pg_stat_activity
WHERE backend_type = 'client backend'
ORDER BY datname, state;

SELECT
    id,
    file_name,
    status,
    error_description,
    created,
    uploaded
FROM file_upload
ORDER BY id DESC
LIMIT 5;
```
