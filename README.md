```java
WITH latest_request AS (
    SELECT
        mar.*,
        ROW_NUMBER() OVER (
            PARTITION BY mar.initiative_metric_assignment_id
            ORDER BY mar.created_at DESC, mar.id DESC
        ) AS rn
    FROM metric_applicability_request mar
)
SELECT
    imt.ai_agent_id AS initiative_id,
    imt.agent_type,
    ima.id AS assignment_id,
    ima.metric_id,
    md.name AS metric_name,
    ima.applicability_status,

    lr.id AS request_id,
    lr.status AS request_status,
    lr.resume_period,
    lr.effective_from_period,
    lr.effective_to_period,
    lr.is_visible_in_office,
    lr.created_at,
    lr.updated_at,

    CASE
        /*
         * APPROVE с resumePeriod обязан сразу записать
         * effectiveToPeriod = resumePeriod.
         */
        WHEN lr.status = 'APPROVED'
             AND lr.resume_period IS NOT NULL
             AND lr.effective_to_period IS NULL
        THEN 'BROKEN: APPROVED + resume_period, но effective_to_period NULL'

        /*
         * Открытая APPROVED-заявка означает, что метрика
         * сейчас неприменима. Assignment не должен быть ACTIVE.
         */
        WHEN ima.applicability_status = 'ACTIVE'
             AND lr.status = 'APPROVED'
             AND lr.effective_to_period IS NULL
        THEN 'BROKEN: ACTIVE assignment + открытый APPROVED request'

        /*
         * Видимая PENDING-заявка — текущий процесс согласования.
         * Assignment при этом должен быть PENDING.
         */
        WHEN ima.applicability_status = 'ACTIVE'
             AND lr.status = 'PENDING'
             AND lr.is_visible_in_office = true
        THEN 'BROKEN: ACTIVE assignment + активный PENDING request'

        /*
         * Если assignment сейчас PENDING,
         * последняя заявка должна быть активной PENDING.
         */
        WHEN ima.applicability_status = 'PENDING'
             AND (
                 lr.id IS NULL
                 OR lr.status <> 'PENDING'
                 OR lr.is_visible_in_office <> true
             )
        THEN 'BROKEN: PENDING assignment без актуальной PENDING-заявки'

        /*
         * NOT_APPLICABLE должен соответствовать
         * согласованной APPROVED-заявке.
         */
        WHEN ima.applicability_status = 'NOT_APPLICABLE'
             AND (
                 lr.id IS NULL
                 OR lr.status <> 'APPROVED'
             )
        THEN 'BROKEN: NOT_APPLICABLE assignment без актуальной APPROVED-заявки'

        ELSE 'OK'
    END AS data_state

FROM initiative_metric_assignment ima
JOIN initiative_metric_type imt
    ON imt.id = ima.initiative_agent_type_id
JOIN metrics_directory md
    ON md.id = ima.metric_id
LEFT JOIN latest_request lr
    ON lr.initiative_metric_assignment_id = ima.id
   AND lr.rn = 1

WHERE
       (
           lr.status = 'APPROVED'
           AND lr.resume_period IS NOT NULL
           AND lr.effective_to_period IS NULL
       )
    OR (
           ima.applicability_status = 'ACTIVE'
           AND lr.status = 'APPROVED'
           AND lr.effective_to_period IS NULL
       )
    OR (
           ima.applicability_status = 'ACTIVE'
           AND lr.status = 'PENDING'
           AND lr.is_visible_in_office = true
       )
    OR (
           ima.applicability_status = 'PENDING'
           AND (
               lr.id IS NULL
               OR lr.status <> 'PENDING'
               OR lr.is_visible_in_office <> true
           )
       )
    OR (
           ima.applicability_status = 'NOT_APPLICABLE'
           AND (
               lr.id IS NULL
               OR lr.status <> 'APPROVED'
           )
       )

ORDER BY imt.ai_agent_id, ima.id;




WITH params AS (
    SELECT
        (date_trunc('month', current_date) - interval '1 month')::date AS saving_period
)
SELECT
    p.saving_period,

    imt.ai_agent_id AS initiative_id,
    imt.agent_type,

    ima.id AS assignment_id,
    ima.metric_id,
    md.name AS metric_name,
    ima.applicability_status,

    mar.id AS blocking_request_id,
    mar.status AS blocking_request_status,
    mar.effective_from_period,
    mar.effective_to_period,
    mar.resume_period,
    mar.is_visible_in_office,
    mar.created_at,
    mar.updated_at

FROM params p
JOIN initiative_metric_assignment ima
    ON true
JOIN initiative_metric_type imt
    ON imt.id = ima.initiative_agent_type_id
JOIN metrics_directory md
    ON md.id = ima.metric_id
JOIN metric_applicability_request mar
    ON mar.initiative_metric_assignment_id = ima.id

WHERE mar.status IN ('PENDING', 'APPROVED')
  AND mar.effective_from_period <= p.saving_period
  AND (
      mar.effective_to_period IS NULL
      OR p.saving_period < mar.effective_to_period
  )

ORDER BY imt.ai_agent_id, ima.id, mar.created_at DESC, mar.id DESC;
```
