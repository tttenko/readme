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
    obj.id,
    obj.initiative_id,
    obj.crossgoal,
    obj.ci_solution,
    obj.name,
    obj.is_agent,
    obj.gigausage,
    obj.meta_link,
    obj.source,
    obj.is_active
FROM initiative_meta_ai_object obj
WHERE obj.crossgoal = 'CROSSGOAL-41438'
  AND obj.ci_solution IN (
      'KE-TEST-001',
      'KE-TEST-002',
      'KE-TEST-003'
  )
ORDER BY obj.initiative_id, obj.ci_solution;

SELECT
    id,
    ai_agent_id,
    file_upload_id,
    changed_objects_count,
    event_type,
    created_at
FROM initiative_ai_object_history
WHERE file_upload_id = 118
ORDER BY ai_agent_id;

SELECT
    initiative_id,
    COUNT(*) AS objects_count
FROM initiative_meta_ai_object
WHERE crossgoal = 'CROSSGOAL-41438'
  AND ci_solution IN (
      'KE-TEST-001',
      'KE-TEST-002',
      'KE-TEST-003'
  )
GROUP BY initiative_id
ORDER BY initiative_id;

```
