```java


SELECT
    a.id AS initiative_id,
    a.agent_name,
    a.disabled,
    COUNT(ife.id) AS financial_effect_count
FROM ai_agent a
JOIN initiative_financial_effect ife
    ON ife.initiative_id = a.id
WHERE ife.effect_year = 2026
  AND ife.disabled = false
  AND a.disabled = false
GROUP BY a.id, a.agent_name, a.disabled
ORDER BY financial_effect_count DESC;
```
