```java
BEGIN;

UPDATE initiative_meta_ai_object
SET is_active = false
WHERE id = 1596;

-- Здесь GET нужно отправить из Insomnia.

ROLLBACK;
```
