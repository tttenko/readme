```java

BEGIN;

WITH candidate_pool AS (
    SELECT
        imt.id         AS initiative_agent_type_id,
        imt.ai_agent_id AS initiative_id,
        imt.agent_type AS agent_type,
        md.id          AS metric_id
    FROM prm_ai.initiative_metric_type imt
    CROSS JOIN prm_ai.metrics_directory md
    WHERE md.is_active = TRUE
      AND NOT EXISTS (
          SELECT 1
          FROM prm_ai.initiative_metric_assignment ima
          WHERE ima.initiative_agent_type_id = imt.id
            AND ima.metric_id = md.id
      )
),
selected AS (
    SELECT
        initiative_agent_type_id,
        initiative_id,
        agent_type,
        metric_id,
        ROW_NUMBER() OVER (
            ORDER BY initiative_agent_type_id, metric_id
        ) AS rn
    FROM candidate_pool
    ORDER BY initiative_agent_type_id, metric_id
    LIMIT 2
),
inserted_assignments AS (
    INSERT INTO prm_ai.initiative_metric_assignment (
        initiative_agent_type_id,
        metric_id,
        applicability_status
    )
    SELECT
        initiative_agent_type_id,
        metric_id,
        'PENDING'
    FROM selected
    RETURNING
        id,
        initiative_agent_type_id,
        metric_id
),
test_assignments AS (
    SELECT
        ia.id AS assignment_id,
        s.initiative_id,
        s.agent_type,
        s.metric_id,
        s.rn
    FROM inserted_assignments ia
    JOIN selected s
      ON s.initiative_agent_type_id = ia.initiative_agent_type_id
     AND s.metric_id = ia.metric_id
),
inserted_requests AS (
    INSERT INTO prm_ai.metric_applicability_request (
        initiative_metric_assignment_id,
        status,
        comment,
        resume_period,
        is_visible_in_office,
        effective_from_period,
        effective_to_period,
        created_by,
        decision_by,
        created_at,
        updated_at
    )
    SELECT
        assignment_id,
        'PENDING',

        CASE rn
            WHEN 1 THEN 'PATCH_TEST_20260917_APPROVE'
            WHEN 2 THEN 'PATCH_TEST_20260917_REJECT'
        END,

        CASE rn
            WHEN 1 THEN
                (
                    date_trunc('month', current_date)
                    + interval '3 months'
                )::date
            ELSE NULL
        END,

        TRUE,
        date_trunc('month', current_date)::date,
        NULL,

        -- Для проверки только бизнес-логики можно оставить 0.
        -- Для проверки email поставь сюда id реального пользователя prm-auth.
        0,

        NULL,
        current_date,
        current_timestamp

    FROM test_assignments

    RETURNING
        id,
        initiative_metric_assignment_id,
        comment
),
inserted_history AS (
    INSERT INTO prm_ai.metric_applicability_history (
        metric_applicability_request_id,
        action,
        created_by,
        comment,
        created_at
    )
    SELECT
        ir.id,
        'REQUEST_CREATED',
        '0',
        ir.comment,
        current_date
    FROM inserted_requests ir

    RETURNING id
)
SELECT
    CASE ta.rn
        WHEN 1 THEN 'APPROVE -> CANCEL_DECISION'
        WHEN 2 THEN 'REJECT -> CANCEL_DECISION'
    END AS test_scenario,

    ta.initiative_id,
    ta.metric_id,
    ta.agent_type,
    ta.assignment_id,
    ir.id AS request_id,

    'PENDING' AS request_status,
    'PENDING' AS applicability_status

FROM inserted_requests ir
JOIN test_assignments ta
  ON ta.assignment_id = ir.initiative_metric_assignment_id

ORDER BY ta.rn;

COMMIT;

```
