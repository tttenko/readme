```java

@Query(
    value = """
        SELECT DISTINCT a.id
        FROM ai_agent a
        LEFT JOIN status s ON a.agent_status_id = s.id
        LEFT JOIN division d ON a.division_id = d.id
        LEFT JOIN block b1 ON d.block_id = b1.id
        LEFT JOIN block b2 ON a.block_id = b2.id
        LEFT JOIN implemented_platform ip ON a.id = ip.ai_agent_id
        LEFT JOIN platform p ON ip.platform_id = p.id
        LEFT JOIN program prgm ON a.program_id = prgm.id
        LEFT JOIN initiative_type ite
            ON a.agent_initiative_type = ite.code
        LEFT JOIN user_audience ua
            ON a.user_audience_code = ua.code
        LEFT JOIN agent_contact ac ON a.id = ac.agent_id
        WHERE a.disabled = :disabled
          AND (
              :search IS NULL
              OR LOWER(a.agent_name)
                  LIKE LOWER(CONCAT('%', :search, '%'))
              OR LOWER(a.agent_id)
                  LIKE LOWER(CONCAT('%', :search, '%'))
              OR LOWER(a.agent_jira_url)
                  LIKE LOWER(CONCAT('%', :search, '%'))
          )
          AND (:terbankIds IS NULL OR a.terbank_id IN (:terbankIds))
          AND (
              COALESCE(:programCodes) IS NULL
              OR prgm.code IN (:programCodes)
          )
          AND (
              COALESCE(:agentStatusCodes) IS NULL
              OR s.code IN (:agentStatusCodes)
          )
          AND (
              COALESCE(:blockCodes) IS NULL
              OR b2.code IN (:blockCodes)
              OR b1.code IN (:blockCodes)
          )
          AND (
              COALESCE(:divisionCodes) IS NULL
              OR d.code IN (:divisionCodes)
          )
          AND (
              COALESCE(:platformCodes) IS NULL
              OR p.code IN (:platformCodes)
          )
          AND (
              COALESCE(:initiativeTypes) IS NULL
              OR ite.code IN (:initiativeTypes)
          )
          AND (
              COALESCE(:userAudiences) IS NULL
              OR ua.code IN (:userAudiences)
          )
          AND (
              COALESCE(:deadlineExpiredIds) IS NULL
              OR a.deadline_expired IN (:deadlineExpiredIds)
          )
          AND (
              :userId IS NULL
              OR a.owner_id = :userId
              OR ac.user_id = :userId
          )
    """,
    nativeQuery = true,
)
fun findFilteredInitiativeIds(
    @Param("search")
    search: String?,

    @Param("terbankIds")
    terbankIds: Collection<Long>?,

    @Param("programCodes")
    programCodes: Collection<String>?,

    @Param("agentStatusCodes")
    agentStatusCodes: Collection<String>?,

    @Param("blockCodes")
    blockCodes: Collection<String>?,

    @Param("divisionCodes")
    divisionCodes: Collection<String>?,

    @Param("platformCodes")
    platformCodes: Collection<String>?,

    @Param("initiativeTypes")
    initiativeTypes: Collection<String>?,

    @Param("userAudiences")
    userAudiences: Collection<String>?,

    @Param("disabled")
    disabled: Boolean,

    @Param("deadlineExpiredIds")
    deadlineExpiredIds: Collection<String>?,

    @Param("userId")
    userId: Long?,
): Set<Long>

В основном findAll добавляются параметры:

@Param("filterByDeviation")
filterByDeviation: Boolean,

@Param("deviationInitiativeIds")
deviationInitiativeIds: Collection<Long>,

И в основной запрос и countQuery добавляется:

AND (
    :filterByDeviation = false
    OR a.id IN (:deviationInitiativeIds)
)





```
