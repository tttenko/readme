```java


SELECT
    id,
    initiative_id,
    effect_year,
    effect_year::text AS year_text,
    type_code,
    plan_value,
    fact_value,
    created_at
FROM initiative_financial_effect
ORDER BY created_at DESC;
```
