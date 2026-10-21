```java

SELECT
    column_name,
    data_type,
    is_nullable
FROM information_schema.columns
WHERE table_name = 'initiative_financial_effect'
  AND column_name IN ('plan_value', 'fact_value');

```
