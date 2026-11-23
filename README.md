```java


SELECT
    id,
    file_name,
    status,
    error_description,
    created,
    uploaded
FROM file_upload
WHERE id = 118;

SELECT
    COUNT(*) AS history_count,
    COALESCE(SUM(changed_objects_count), 0) AS changed_objects_count
FROM initiative_ai_object_history
WHERE file_upload_id = :newFileId;
```
