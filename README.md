```java
SELECT
    r.id AS request_id,
    r.status AS request_status,
    a.applicability_status,
    r.decision_by,
    r.effective_to_period,
    r.is_visible_in_office,
    r.updated_at
FROM prm_ai.metric_applicability_request r
JOIN prm_ai.initiative_metric_assignment a
    ON a.id = r.initiative_metric_assignment_id
WHERE r.id = 15;

Ожидаем:

request_status        = PENDING
applicability_status  = PENDING
decision_by           = НЕ изменился
effective_to_period   = null
is_visible_in_office  = true

И история:

SELECT
    h.id,
    h.action,
    h.created_by,
    h.comment,
    h.created_at
FROM prm_ai.metric_applicability_history h
WHERE h.metric_applicability_request_id = 15
ORDER BY h.created_at, h.id;

Должно стать:

REQUEST_CREATED
APPROVED
CANCEL_DECISION

```
