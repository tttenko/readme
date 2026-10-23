```java
UPDATE upload_permission
SET upload_allowed = TRUE,
    last_updated = NOW(),
    updated_by = 1
WHERE id = 'GLOBAL';


```
