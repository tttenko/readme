```java
OR

(
    :#{#params.stageDeadlineTodayEnabled} = true
    AND EXISTS (
        SELECT 1
        FROM agent_status_sla sla
        WHERE sla.ai_agent_id = candidate.id
          AND sla.completed_date IS NULL
          AND sla.planned_date >= :#{#params.todayStart}
          AND sla.planned_date < :#{#params.tomorrowStart}
    )
)
  ```
