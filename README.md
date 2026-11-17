```java
SELECT
    ima.id AS assignment_id,
    ima.applicability_status,
    mar.id AS request_id,
    mar.status,
    mar.resume_period,
    mar.effective_from_period,
    mar.effective_to_period,
    mar.is_visible_in_office,
    mar.created_at,
    mar.updated_at
FROM initiative_metric_assignment ima
JOIN metric_applicability_request mar
    ON mar.initiative_metric_assignment_id = ima.id
WHERE ima.id = 26
ORDER BY mar.created_at DESC, mar.id DESC;

```
