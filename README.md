```java
Временно отключи release
UPDATE prm_ai.status
SET disabled = TRUE
WHERE code = 'release'
RETURNING
    id,
    code,
    ordering,
    disabled;

Если автокоммит выключен:

COMMIT;

Проверь:

SELECT
    id,
    code,
    ordering,
    disabled
FROM prm_ai.status
WHERE code = 'release';
  ```
