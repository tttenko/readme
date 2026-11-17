```java
WITH params AS (
    SELECT
        733::BIGINT AS initiative_id,
        (date_trunc('month', current_date) - interval '1 month')::date AS saving_period
)
SELECT
    p.saving_period,

    imt.id AS initiative_metric_type_id,
    imt.agent_type,

    ima.id AS assignment_id,
    ima.metric_id,
    md.name AS metric_name,
    ima.applicability_status,

    r.id AS request_id,
    r.status AS request_status,
    r.resume_period,
    r.effective_from_period,
    r.effective_to_period,
    r.is_visible_in_office,
    r.created_at,
    r.updated_at,

    CASE
        WHEN r.id IS NULL THEN false
        WHEN r.status IN ('PENDING', 'APPROVED')
             AND r.effective_from_period <= p.saving_period
             AND (
                 r.effective_to_period IS NULL
                 OR p.saving_period < r.effective_to_period
             )
        THEN true
        ELSE false
    END AS should_block_for_saving_period,

    CASE
        WHEN ima.applicability_status <> 'ACTIVE'
        THEN true
        ELSE false
    END AS current_code_likely_blocks

FROM params p

JOIN initiative_metric_type imt
    ON imt.ai_agent_id = p.initiative_id

JOIN initiative_metric_assignment ima
    ON ima.initiative_agent_type_id = imt.id

JOIN metrics_directory md
    ON md.id = ima.metric_id

LEFT JOIN LATERAL (
    SELECT mar.*
    FROM metric_applicability_request mar
    WHERE mar.initiative_metric_assignment_id = ima.id
    ORDER BY mar.created_at DESC, mar.id DESC
    LIMIT 1
) r ON true

ORDER BY
    imt.agent_type,
    md.name;
