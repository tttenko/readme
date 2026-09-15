```java
SELECT
    request.id AS request_id,
    ai_agent.id AS initiative_id,
    ai_agent.agent_name AS initiative_name,
    metric.name AS metric_name,
    initiative_metric_type.agent_type,
    request.status AS request_status,
    assignment.applicability_status,
    request.is_visible_in_office,
    request.comment,
    request.created_at
FROM metric_applicability_request request
JOIN initiative_metric_assignment assignment
    ON assignment.id = request.initiative_metric_assignment_id
JOIN initiative_metric_type
    ON initiative_metric_type.id = assignment.initiative_agent_type_id
JOIN ai_agent
    ON ai_agent.id = initiative_metric_type.ai_agent_id
JOIN metrics_directory metric
    ON metric.id = assignment.metric_id
WHERE request.comment LIKE 'TEST_QUEUE_%'
ORDER BY request.created_at DESC, request.id DESC;
```
