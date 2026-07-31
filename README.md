```java
data class InitiativeDeviationCountResponse(

    @field:Schema(
        description = "Количество инициатив с отклонениями",
        example = "12",
    )
    val totalInitiativeWithDeviations: Long,
)

data class InitiativeDeviationCountQueryParams(
    val userId: Long?,

    val todayStart: LocalDateTime,
    val tomorrowStart: LocalDateTime,
    val inTwoDaysStart: LocalDateTime,
    val inThreeDaysStart: LocalDateTime,
    val inFourDaysStart: LocalDateTime,

    val currentPeriod: LocalDate,
    val gigaUsageProject: String,

    val stageDeadlineExpiredEnabled: Boolean,
    val stageDeadlinesNotFilledEnabled: Boolean,
    val gigaUsageNotFilledEnabled: Boolean,
    val enablersNotFilledEnabled: Boolean,
    val stageDeadlineTomorrowEnabled: Boolean,
    val stageDeadlineInTwoDaysEnabled: Boolean,
    val stageDeadlineInThreeDaysEnabled: Boolean,
    val checkCurrentPeriodMetrics: Boolean,
)

interface InitiativeDeviationCountRepository :
    Repository<AIAgentEntity, Long> {

    @Query(
        nativeQuery = true,
        value = """
            WITH candidates AS (
                SELECT agent.id
                FROM ai_agent agent
                WHERE agent.disabled = false
                  AND (
                      CAST(:#{#params.userId} AS BIGINT) IS NULL
                      OR agent.owner_id = :#{#params.userId}
                      OR EXISTS (
                          SELECT 1
                          FROM agent_contact contact
                          WHERE contact.agent_id = agent.id
                            AND contact.user_id = :#{#params.userId}
                      )
                  )
            ),

            cheap_deviations AS (
                SELECT candidate.id
                FROM candidates candidate
                WHERE
                    (
                        :#{#params.stageDeadlineExpiredEnabled} = true
                        AND EXISTS (
                            SELECT 1
                            FROM agent_status_sla sla
                            WHERE sla.ai_agent_id = candidate.id
                              AND sla.completed_date IS NULL
                              AND sla.planned_date IS NOT NULL
                              AND sla.planned_date < :#{#params.todayStart}
                        )
                    )

                    OR

                    (
                        :#{#params.stageDeadlinesNotFilledEnabled} = true
                        AND EXISTS (
                            SELECT 1
                            FROM agent_status_sla sla
                            WHERE sla.ai_agent_id = candidate.id
                              AND sla.completed_date IS NULL
                              AND sla.planned_date IS NULL
                        )
                    )

                    OR

                    (
                        :#{#params.gigaUsageNotFilledEnabled} = true
                        AND NOT EXISTS (
                            SELECT 1
                            FROM jira_issue jira
                            WHERE jira.agent_id = candidate.id
                              AND LOWER(jira.project) =
                                  LOWER(:#{#params.gigaUsageProject})
                              AND NULLIF(BTRIM(jira.jira_key), '') IS NOT NULL
                        )
                    )

                    OR

                    (
                        :#{#params.enablersNotFilledEnabled} = true
                        AND NOT EXISTS (
                            SELECT 1
                            FROM agent_enabler agent_enabler
                            WHERE agent_enabler.agent_id = candidate.id
                        )
                    )

                    OR

                    (
                        :#{#params.stageDeadlineTomorrowEnabled} = true
                        AND EXISTS (
                            SELECT 1
                            FROM agent_status_sla sla
                            WHERE sla.ai_agent_id = candidate.id
                              AND sla.completed_date IS NULL
                              AND sla.planned_date >=
                                  :#{#params.tomorrowStart}
                              AND sla.planned_date <
                                  :#{#params.inTwoDaysStart}
                        )
                    )

                    OR

                    (
                        :#{#params.stageDeadlineInTwoDaysEnabled} = true
                        AND EXISTS (
                            SELECT 1
                            FROM agent_status_sla sla
                            WHERE sla.ai_agent_id = candidate.id
                              AND sla.completed_date IS NULL
                              AND sla.planned_date >=
                                  :#{#params.inTwoDaysStart}
                              AND sla.planned_date <
                                  :#{#params.inThreeDaysStart}
                        )
                    )

                    OR

                    (
                        :#{#params.stageDeadlineInThreeDaysEnabled} = true
                        AND EXISTS (
                            SELECT 1
                            FROM agent_status_sla sla
                            WHERE sla.ai_agent_id = candidate.id
                              AND sla.completed_date IS NULL
                              AND sla.planned_date >=
                                  :#{#params.inThreeDaysStart}
                              AND sla.planned_date <
                                  :#{#params.inFourDaysStart}
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
                WHERE :#{#params.checkCurrentPeriodMetrics} = true
                  AND EXISTS (
                      SELECT 1
                      FROM initiative_metric_type metric_type
                      INNER JOIN metrics_directory metric
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
                            FROM initiative_metric_value metric_value
                            WHERE metric_value.initiative_agent_type_id =
                                  metric_type.id
                              AND metric_value.metric_directory_id =
                                  metric.id
                              AND metric_value.period_month =
                                  :#{#params.currentPeriod}
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
            ) initiatives_with_deviations
        """,
    )
    fun countInitiativesWithDeviations(
        @Param("params")
        params: InitiativeDeviationCountQueryParams,
    ): Long
}

@Service
class InitiativeDeviationCountService(
    private val initiativeDeviationCountRepository:
        InitiativeDeviationCountRepository,
    private val dateTimeProvider: DateTimeProvider,
    private val properties: InitiativeDeviationProperties,
) {

    @Transactional(readOnly = true)
    fun getInitiativesWithDeviationsCount(
        userId: Long?,
    ): InitiativeDeviationCountResponse {
        val today = dateTimeProvider.currentDate()

        val params = InitiativeDeviationCountQueryParams(
            userId = userId,

            todayStart = today.atStartOfDay(),
            tomorrowStart = today.plusDays(1).atStartOfDay(),
            inTwoDaysStart = today.plusDays(2).atStartOfDay(),
            inThreeDaysStart = today.plusDays(3).atStartOfDay(),
            inFourDaysStart = today.plusDays(4).atStartOfDay(),

            currentPeriod = today.withDayOfMonth(1),
            gigaUsageProject = GIGAUSAGE_PROJECT,

            stageDeadlineExpiredEnabled =
                properties.getRequiredRule(
                    InitiativeDeviationCode.STAGE_DEADLINE_EXPIRED,
                ).enabled,

            stageDeadlinesNotFilledEnabled =
                properties.getRequiredRule(
                    InitiativeDeviationCode.STAGE_DEADLINES_NOT_FILLED,
                ).enabled,

            gigaUsageNotFilledEnabled =
                properties.getRequiredRule(
                    InitiativeDeviationCode.GIGAUSAGE_NOT_FILLED,
                ).enabled,

            enablersNotFilledEnabled =
                properties.getRequiredRule(
                    InitiativeDeviationCode.ENABLERS_NOT_FILLED,
                ).enabled,

            stageDeadlineTomorrowEnabled =
                properties.getRequiredRule(
                    InitiativeDeviationCode.STAGE_DEADLINE_TOMORROW,
                ).enabled,

            stageDeadlineInTwoDaysEnabled =
                properties.getRequiredRule(
                    InitiativeDeviationCode.STAGE_DEADLINE_IN_2_DAYS,
                ).enabled,

            stageDeadlineInThreeDaysEnabled =
                properties.getRequiredRule(
                    InitiativeDeviationCode.STAGE_DEADLINE_IN_3_DAYS,
                ).enabled,

            checkCurrentPeriodMetrics =
                properties.getRequiredRule(
                    InitiativeDeviationCode
                        .CURRENT_PERIOD_METRICS_NOT_FILLED,
                ).enabled &&
                    today.dayOfMonth >
                    properties.currentPeriodMetricsDeadlineDay,
        )

        val count =
            initiativeDeviationCountRepository
                .countInitiativesWithDeviations(params)

        return InitiativeDeviationCountResponse(
            totalInitiativeWithDeviations = count,
        )
    }

    private companion object {
        const val GIGAUSAGE_PROJECT = "gigausage"
    }
}

@GetMapping("/initiative/deviations/count")
@PreAuthorize(
    "hasAnyAuthority(" +
        "'PROJECT_OFFICE', " +
        "'CMS_ADMIN', " +
        "'TRANSFORMATION_OFFICE'" +
        ")"
)
@Operation(
    summary = "Получение количества инициатив с отклонениями",
    responses = [
        ApiResponse(
            responseCode = "200",
            description = "Количество инициатив успешно рассчитано",
            content = [
                Content(
                    mediaType = "application/json",
                    schema = Schema(
                        implementation =
                            InitiativeDeviationCountResponse::class,
                    ),
                ),
            ],
        ),
        ApiResponse(
            responseCode = "403",
            description = "Недостаточно прав",
        ),
    ],
)
fun getInitiativesWithDeviationsCount(
    @Parameter(
        description = "Идентификатор пользователя",
        example = "23561",
        required = false,
    )
    @RequestParam(
        name = "userId",
        required = false,
    )
    userId: Long?,
): InitiativeDeviationCountResponse =
    initiativeDeviationCountService
        .getInitiativesWithDeviationsCount(
            userId = userId,
        )
  ```
