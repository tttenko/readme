```java
WITH payload(metric_id) AS (
    VALUES
        ('a683e462-d7b9-4591-a7ae-11db2d6536d2'::uuid),
        ('10000000-0000-0000-0000-000000000002'::uuid),
        ('360b0399-f783-40d5-aa22-4830d0dcf602'::uuid),
        ('10000000-0000-0000-0000-000000000003'::uuid),
        ('05729d34-7a94-42a7-9d44-de6250cf2fac'::uuid),
        ('10000000-0000-0000-0000-000000000004'::uuid),
        ('32651ab3-154d-4415-9d76-998ca3467a09'::uuid),
        ('657d4617-557f-4f63-b506-147cfd73302d'::uuid),
        ('ee5e8977-8120-40cc-8a4f-d77604320fd6'::uuid),
        ('5a91b836-d4c8-4fe8-84b4-92921a36ebd4'::uuid)
)
SELECT
    p.metric_id,
    md.name,
    ima.id AS assignment_id,
    ima.applicability_status,
    mar.id AS request_id,
    mar.status AS request_status,
    mar.effective_from_period,
    mar.effective_to_period,
    mar.resume_period,
    mar.is_visible_in_office
FROM payload p
LEFT JOIN metrics_directory md
    ON md.id = p.metric_id
LEFT JOIN initiative_metric_type imt
    ON imt.ai_agent_id = 733
   AND imt.agent_type = 'autonomous'
LEFT JOIN initiative_metric_assignment ima
    ON ima.initiative_agent_type_id = imt.id
   AND ima.metric_id = p.metric_id
LEFT JOIN metric_applicability_request mar
    ON mar.initiative_metric_assignment_id = ima.id
ORDER BY p.metric_id, mar.created_at DESC, mar.id DESC;

```
