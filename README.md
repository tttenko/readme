```java


SELECT
    COUNT(*) AS history_count,
    COALESCE(SUM(changed_objects_count), 0) AS changed_objects_count
FROM initiative_ai_object_history
WHERE file_upload_id = 121;
```
