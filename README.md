```java

BEGIN;

-- 1. Разрешаем повторную загрузку Excel
UPDATE upload_permission
SET upload_allowed = TRUE,
    last_updated = NOW(),
    updated_by = 1
WHERE id = 'GLOBAL';

-- 2. Меняем статус загрузки №94 на success
UPDATE file_upload
SET status = 'success',
    error_description = NULL,
    updated = NOW(),
    uploaded = NOW(),
    updated_by = 1
WHERE id = 94;

COMMIT;

```
