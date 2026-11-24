```java
/**
     * Возвращает ID CROSSGOAL-инициатив на применимых стадиях,
     * у которых отсутствует хотя бы один активный AI-объект.
     *
     * Проверка выполняется одним запросом для набора инициатив.
     */
    @Query(
        value = """
            SELECT agent.id
            FROM ai_agent agent
            INNER JOIN status current_status
                ON current_status.id = agent.agent_status_id
            WHERE agent.id IN (:initiativeIds)
              AND current_status.code IN (:applicableStatusCodes)
              AND EXISTS (
                  SELECT 1
                  FROM jira_issue jira
                  WHERE jira.agent_id = agent.id
                    AND LOWER(jira.project) = LOWER(:crossgoalProject)
                    AND LOWER(jira.type) = 'initiative'
                    AND NULLIF(BTRIM(jira.jira_key), '') IS NOT NULL
              )
              AND NOT EXISTS (
                  SELECT 1
                  FROM initiative_meta_ai_object ai_object
                  WHERE ai_object.initiative_id = agent.id
                    AND ai_object.is_active IS TRUE
              )
        """,
        nativeQuery = true
    )
    fun findInitiativeIdsWithoutActiveAiObjects(
        @Param("initiativeIds") initiativeIds: Collection<Long>,
        @Param("applicableStatusCodes") applicableStatusCodes: Collection<String>,
        @Param("crossgoalProject") crossgoalProject: String
    ): List<Long>
    
```
