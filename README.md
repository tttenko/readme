```java
SELECT
    obj.id,
    obj.initiative_id,
    obj.crossgoal,
    obj.ci_solution,
    obj.name,
    obj.is_agent,
    obj.gigausage,
    obj.meta_link,
    obj.is_active
FROM initiative_meta_ai_object obj
WHERE obj.initiative_id = 1647
  AND obj.crossgoal = 'CROSSGOAL-41438'
  AND obj.is_active = true
ORDER BY obj.id;
```
