```java
SELECT
    imt.ai_agent_id AS initiative_id,
    imt.agent_type,
    ima.id AS assignment_id,
    ima.metric_id,
    md.name AS metric_name,
    ima.applicability_status,

    mar.id AS request_id,
    mar.status AS request_status,
    mar.effective_from_period,
    mar.effective_to_period,
    mar.resume_period,
    mar.is_visible_in_office,
    mar.created_at,
    mar.updated_at

FROM metric_applicability_request mar
JOIN initiative_metric_assignment ima
    ON ima.id = mar.initiative_metric_assignment_id
JOIN initiative_metric_type imt
    ON imt.id = ima.initiative_agent_type_id
JOIN metrics_directory md
    ON md.id = ima.metric_id

WHERE mar.status = 'PENDING'
  AND mar.is_visible_in_office = true
  AND ima.applicability_status <> 'PENDING'

ORDER BY imt.ai_agent_id, ima.id, mar.created_at DESC, mar.id DESC;
```
