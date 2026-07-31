```java
SELECT
    (SELECT COUNT(*) FROM candidates) AS all_candidates,
    (SELECT COUNT(*) FROM cheap_deviations) AS cheap_deviations_count,
    (SELECT COUNT(*) FROM metric_deviations) AS metric_deviations_count,
    (
        SELECT COUNT(*)
        FROM (
            SELECT id FROM cheap_deviations
            UNION ALL
            SELECT id FROM metric_deviations
        ) deviations
    ) AS total_with_deviations;
  ```
