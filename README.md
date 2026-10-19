```java


WITH initiative_deviations AS (
    SELECT
        agent.id,
        agent.agent_id,
        agent.agent_name,
        status.code AS status_code,

        -- Просрочен дедлайн этапа (priority = 1)
        EXISTS (
            SELECT 1
            FROM agent_status_sla sla
            JOIN status stage ON stage.id = sla.agent_status_id
            WHERE sla.ai_agent_id = agent.id
              AND sla.completed_date IS NULL
              AND stage.disabled = false
              AND stage.ordering >= status.ordering
              AND sla.planned_date::date <
                  (NOW() AT TIME ZONE 'Europe/Moscow')::date
        ) AS deadline_expired,

        -- Не заполнены дедлайны этапов (priority = 3)
        EXISTS (
            SELECT 1
            FROM status stage
            LEFT JOIN agent_status_sla sla
                ON sla.agent_status_id = stage.id
                AND sla.ai_agent_id = agent.id
            WHERE stage.disabled = false
              AND stage.ordering >= status.ordering
              AND (
                  sla.ai_agent_id IS NULL
                  OR (
                      sla.completed_date IS NULL
                      AND sla.planned_date IS NULL
                  )
              )
        ) AS deadlines_not_filled,

        -- Не заполнены энейблеры (priority = 5)
        NOT EXISTS (
            SELECT 1
            FROM agent_enabler enabler
            WHERE enabler.agent_id = agent.id
        ) AS enablers_not_filled

    FROM ai_agent agent
    LEFT JOIN status ON status.id = agent.agent_status_id
    WHERE agent.disabled = false
)
SELECT
    id,
    agent_id,
    agent_name,
    status_code,
    deadline_expired,
    deadlines_not_filled,
    enablers_not_filled
FROM initiative_deviations
WHERE deadline_expired
   OR deadlines_not_filled
   OR enablers_not_filled
ORDER BY
    (deadline_expired::int +
     deadlines_not_filled::int +
     enablers_not_filled::int) DESC,
    id
LIMIT 20;


```
