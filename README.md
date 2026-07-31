```java
EXPLAIN (
    ANALYZE,
    BUFFERS,
    VERBOSE,
    SETTINGS
)
WITH params AS (
    SELECT
        23561::BIGINT AS user_id,
        -- Для проверки всех инициатив банка:
        -- NULL::BIGINT AS user_id,

        TRUE AS stage_deadline_expired_enabled,
        TRUE AS stage_deadlines_not_filled_enabled,
        TRUE AS giga_usage_not_filled_enabled,
        TRUE AS enablers_not_filled_enabled,
        TRUE AS stage_deadline_tomorrow_enabled,
        TRUE AS stage_deadline_in_two_days_enabled,
        TRUE AS stage_deadline_in_three_days_enabled,
        TRUE AS check_current_period_metrics,

        TIMESTAMP '2026-07-31 00:00:00' AS today_start,
        TIMESTAMP '2026-08-01 00:00:00' AS tomorrow_start,
        TIMESTAMP '2026-08-02 00:00:00' AS in_two_days_start,
        TIMESTAMP '2026-08-03 00:00:00' AS in_three_days_start,
        TIMESTAMP '2026-08-04 00:00:00' AS in_four_days_start,

        DATE '2026-07-01' AS current_period,
        'gigausage'::VARCHAR AS giga_usage_project
),

candidates AS (
    SELECT agent.id
    FROM prm_ai.ai_agent agent
    CROSS JOIN params p
    WHERE agent.disabled = FALSE
      AND (
          p.user_id IS NULL
          OR agent.owner_id = p.user_id
          OR EXISTS (
              SELECT 1
              FROM prm_ai.agent_contact contact
              WHERE contact.agent_id = agent.id
                AND contact.user_id = p.user_id
          )
      )
),

cheap_deviations AS (
    SELECT candidate.id
    FROM candidates candidate
    CROSS JOIN params p
    WHERE
        (
            p.stage_deadline_expired_enabled
            AND EXISTS (
                SELECT 1
                FROM prm_ai.agent_status_sla sla
                WHERE sla.ai_agent_id = candidate.id
                  AND sla.completed_date IS NULL
                  AND sla.planned_date IS NOT NULL
                  AND sla.planned_date < p.today_start
            )
        )

        OR

        (
            p.stage_deadlines_not_filled_enabled
            AND EXISTS (
                SELECT 1
                FROM prm_ai.agent_status_sla sla
                WHERE sla.ai_agent_id = candidate.id
                  AND sla.completed_date IS NULL
                  AND sla.planned_date IS NULL
            )
        )

        OR

        (
            p.giga_usage_not_filled_enabled
            AND NOT EXISTS (
                SELECT 1
                FROM prm_ai.jira_issue jira
                WHERE jira.agent_id = candidate.id
                  AND LOWER(jira.project) =
                      LOWER(p.giga_usage_project)
                  AND NULLIF(BTRIM(jira.jira_key), '') IS NOT NULL
            )
        )

        OR

        (
            p.enablers_not_filled_enabled
            AND NOT EXISTS (
                SELECT 1
                FROM prm_ai.agent_enabler agent_enabler
                WHERE agent_enabler.agent_id = candidate.id
            )
        )

        OR

        (
            p.stage_deadline_tomorrow_enabled
            AND EXISTS (
                SELECT 1
                FROM prm_ai.agent_status_sla sla
                WHERE sla.ai_agent_id = candidate.id
                  AND sla.completed_date IS NULL
                  AND sla.planned_date >= p.tomorrow_start
                  AND sla.planned_date < p.in_two_days_start
            )
        )

        OR

        (
            p.stage_deadline_in_two_days_enabled
            AND EXISTS (
                SELECT 1
                FROM prm_ai.agent_status_sla sla
                WHERE sla.ai_agent_id = candidate.id
                  AND sla.completed_date IS NULL
                  AND sla.planned_date >= p.in_two_days_start
                  AND sla.planned_date < p.in_three_days_start
            )
        )

        OR

        (
            p.stage_deadline_in_three_days_enabled
            AND EXISTS (
                SELECT 1
                FROM prm_ai.agent_status_sla sla
                WHERE sla.ai_agent_id = candidate.id
                  AND sla.completed_date IS NULL
                  AND sla.planned_date >= p.in_three_days_start
                  AND sla.planned_date < p.in_four_days_start
            )
        )
),

remaining_candidates AS (
    SELECT candidate.id
    FROM candidates candidate
    WHERE NOT EXISTS (
        SELECT 1
        FROM cheap_deviations deviation
        WHERE deviation.id = candidate.id
    )
),

metric_deviations AS (
    SELECT candidate.id
    FROM remaining_candidates candidate
    CROSS JOIN params p
    WHERE p.check_current_period_metrics
      AND EXISTS (
          SELECT 1
          FROM prm_ai.initiative_metric_type metric_type
          INNER JOIN prm_ai.metrics_directory metric
              ON metric.is_active IS TRUE
             AND (
                 (
                     metric_type.agent_type = 'autonomous'
                     AND metric.autonomous_applicability IS TRUE
                 )
                 OR
                 (
                     metric_type.agent_type = 'copilot'
                     AND metric.copilot_applicability IS TRUE
                 )
                 OR
                 (
                     metric_type.agent_type = 'appeals'
                     AND metric.requires_appeals_work IS TRUE
                 )
             )
          WHERE metric_type.ai_agent_id = candidate.id
            AND NOT EXISTS (
                SELECT 1
                FROM prm_ai.initiative_metric_value metric_value
                WHERE metric_value.initiative_agent_type_id =
                      metric_type.id
                  AND metric_value.metric_directory_id =
                      metric.id
                  AND metric_value.period_month =
                      p.current_period
                  AND metric_value.metric_value IS NOT NULL
            )
      )
)

SELECT COUNT(*)
FROM (
    SELECT id
    FROM cheap_deviations

    UNION ALL

    SELECT id
    FROM metric_deviations
) initiatives_with_deviations;
  ```
