```java
@Component
class AiObjectRequiredStrategy(
    private val initiativeMetaAiObjectRepository: InitiativeMetaAiObjectRepository
) : InitiativeDeviationStrategy {

    override val code = InitiativeDeviationCode.AI_OBJECT_REQUIRED

    override val evaluationOrder = 70

    override fun findMatchingInitiativeIds(
        candidateInitiativeIds: Set<Long>,
        session: InitiativeDeviationCalculationSession,
        context: InitiativeDeviationEvaluationContext
    ): Set<Long> {
        if (candidateInitiativeIds.isEmpty()) {
            return emptySet()
        }

        return initiativeMetaAiObjectRepository
            .findInitiativeIdsWithoutActiveAiObjects(
                initiativeIds = candidateInitiativeIds,
                applicableStatusCodes = APPLICABLE_STATUS_CODES,
                crossgoalProject = CROSSGOAL_PROJECT
            )
            .toSet()
    }

    private companion object {
        const val CROSSGOAL_PROJECT = "crossgoal"

        val APPLICABLE_STATUS_CODES = setOf(
            PILOT,
            TARGET_SOLUTION
        )
    }
}

@Component
class InitiativeDeviationCalculatorService(
    private val strategyRegistry: InitiativeDeviationStrategyRegistry,
    private val sessionFactory: InitiativeDeviationCalculationSessionFactory,
    private val properties: InitiativeDeviationProperties,
    private val dateTimeProvider: DateTimeProvider
) {

    fun calculate(initiativeIds: Set<Long>): Map<Long, List<InitiativeDeviationResponse>> {
        if (initiativeIds.isEmpty()) {
            return emptyMap()
        }

        val context = createContext()
        val session = sessionFactory.create(initiativeIds)

        val deviationsByInitiativeId =
            initiativeIds.associateWith { mutableListOf<InitiativeDeviationResponse>() }

        strategyRegistry
            .getEnabledStrategies()
            .forEach { strategy ->
                val matchingIds =
                    strategy.findMatchingInitiativeIds(
                        candidateInitiativeIds = initiativeIds,
                        session = session,
                        context = context
                    )

                val rule = properties.getRequiredRule(strategy.code)

                val response =
                    InitiativeDeviationResponse(
                        code = strategy.code.name,
                        priority = rule.priority,
                        title = rule.title,
                        description = rule.description
                    )

                matchingIds.forEach { initiativeId ->
                    deviationsByInitiativeId
                        .getValue(initiativeId)
                        .add(response)
                }
            }

        return deviationsByInitiativeId.mapValues { (_, deviations) ->
            deviations.sortedBy { deviation -> deviation.priority }
        }
    }

    fun findInitiativeIdsWithAnyDeviation(initiativeIds: Set<Long>): Set<Long> {
        if (initiativeIds.isEmpty()) {
            return emptySet()
        }

        val context = createContext()
        val session = sessionFactory.create(initiativeIds)

        val matchedIds = mutableSetOf<Long>()
        val remainingIds = initiativeIds.toMutableSet()

        strategyRegistry
            .getEnabledStrategies()
            .forEach { strategy ->
                if (remainingIds.isEmpty()) {
                    return@forEach
                }

                val currentMatches =
                    strategy.findMatchingInitiativeIds(
                        candidateInitiativeIds = remainingIds,
                        session = session,
                        context = context
                    )

                matchedIds += currentMatches
                remainingIds -= currentMatches
            }

        return matchedIds
    }

    private fun createContext(): InitiativeDeviationEvaluationContext {
        val today = dateTimeProvider.currentDate()

        return InitiativeDeviationEvaluationContext(
            today = today,
            currentPeriod = YearMonth.from(today).minusMonths(1).atDay(1),
            metricsDeadlineDay = properties.currentPeriodMetricsDeadlineDay
        )
    }
}

@Service
class InitiativeDeviationCountService(
    private val initiativeDeviationCountRepository: InitiativeDeviationCountRepository,
    private val dateTimeProvider: DateTimeProvider,
    private val properties: InitiativeDeviationProperties
) {

    @Transactional(readOnly = true)
    fun getInitiativesWithDeviationsCount(userId: Long?): InitiativeDeviationCountResponse {
        val today = dateTimeProvider.currentDate()

        val params = InitiativeDeviationCountQueryParams(
            userId = userId,
            todayStart = today.atStartOfDay(),
            tomorrowStart = today.plusDays(1).atStartOfDay(),
            inTwoDaysStart = today.plusDays(2).atStartOfDay(),
            inThreeDaysStart = today.plusDays(3).atStartOfDay(),
            inFourDaysStart = today.plusDays(4).atStartOfDay(),
            currentPeriod = YearMonth.from(today).minusMonths(1).atDay(1),
            gigaUsageProject = GIGAUSAGE_PROJECT,
            crossgoalProject = CROSSGOAL_PROJECT,
            pilotStatus = PILOT,
            targetSolutionStatus = TARGET_SOLUTION,
            stageDeadlineExpiredEnabled = properties.getRequiredRule(
                InitiativeDeviationCode.STAGE_DEADLINE_EXPIRED).enabled,
            stageDeadlinesNotFilledEnabled = properties.getRequiredRule(
                InitiativeDeviationCode.STAGE_DEADLINES_NOT_FILLED).enabled,
            gigaUsageNotFilledEnabled = properties.getRequiredRule(
                InitiativeDeviationCode.GIGAUSAGE_NOT_FILLED).enabled,
            enablersNotFilledEnabled = properties.getRequiredRule(
                InitiativeDeviationCode.ENABLERS_NOT_FILLED).enabled,
            stageDeadlineTodayEnabled = properties.getRequiredRule(
                InitiativeDeviationCode.STAGE_DEADLINE_TODAY).enabled,
            stageDeadlineTomorrowEnabled = properties.getRequiredRule(
                InitiativeDeviationCode.STAGE_DEADLINE_TOMORROW).enabled,
            stageDeadlineInTwoDaysEnabled = properties.getRequiredRule(
                InitiativeDeviationCode.STAGE_DEADLINE_IN_2_DAYS).enabled,
            stageDeadlineInThreeDaysEnabled = properties.getRequiredRule(
                InitiativeDeviationCode.STAGE_DEADLINE_IN_3_DAYS).enabled,
            aiObjectRequiredEnabled = properties.getRequiredRule(
                InitiativeDeviationCode.AI_OBJECT_REQUIRED).enabled,
            checkCurrentPeriodMetrics = properties.getRequiredRule(
                InitiativeDeviationCode.CURRENT_PERIOD_METRICS_NOT_FILLED).enabled &&
                    today.dayOfMonth > properties.currentPeriodMetricsDeadlineDay
        )

        val count = initiativeDeviationCountRepository.countInitiativesWithDeviations(params)

        return InitiativeDeviationCountResponse(totalInitiativeWithDeviations = count)
    }

    private companion object {
        const val GIGAUSAGE_PROJECT = "gigausage"
        const val CROSSGOAL_PROJECT = "crossgoal"
    }
}

data class InitiativeDeviationCountQueryParams(
    val userId: Long?,
    val todayStart: LocalDateTime,
    val tomorrowStart: LocalDateTime,
    val inTwoDaysStart: LocalDateTime,
    val inThreeDaysStart: LocalDateTime,
    val inFourDaysStart: LocalDateTime,
    val currentPeriod: LocalDate,
    val gigaUsageProject: String,
    val crossgoalProject: String,
    val pilotStatus: String,
    val targetSolutionStatus: String,
    val stageDeadlineExpiredEnabled: Boolean,
    val stageDeadlinesNotFilledEnabled: Boolean,
    val gigaUsageNotFilledEnabled: Boolean,
    val enablersNotFilledEnabled: Boolean,
    val stageDeadlineTodayEnabled: Boolean,
    val stageDeadlineTomorrowEnabled: Boolean,
    val stageDeadlineInTwoDaysEnabled: Boolean,
    val stageDeadlineInThreeDaysEnabled: Boolean,
    val aiObjectRequiredEnabled: Boolean,
    val checkCurrentPeriodMetrics: Boolean
)

@Repository
interface InitiativeDeviationCountRepository : JpaRepository<AIAgentEntity, Long> {

    @Query(
        nativeQuery = true,
        value = """
            WITH candidates AS (
                SELECT
                    agent.id,
                    current_status.ordering AS current_status_ordering,
                    current_status.code AS current_status_code
                FROM ai_agent agent
                LEFT JOIN status current_status
                    ON current_status.id = agent.agent_status_id
                WHERE agent.disabled = false
                  AND (
                      CAST(:#{#params.userId} AS BIGINT) IS NULL
                      OR agent.owner_id =
                          CAST(:#{#params.userId} AS BIGINT)
                      OR EXISTS (
                          SELECT 1
                          FROM agent_contact contact
                          WHERE contact.agent_id = agent.id
                            AND contact.user_id =
                                CAST(:#{#params.userId} AS BIGINT)
                      )
                  )
            ),

            cheap_deviations AS (
                SELECT candidate.id
                FROM candidates candidate
                WHERE

                    /* STAGE_DEADLINE_EXPIRED */
                    (
                        :#{#params.stageDeadlineExpiredEnabled} = true
                        AND EXISTS (
                            SELECT 1
                            FROM agent_status_sla sla
                            INNER JOIN status stage_status
                                ON stage_status.id = sla.agent_status_id
                            WHERE sla.ai_agent_id = candidate.id
                              AND sla.completed_date IS NULL
                              AND stage_status.disabled = false
                              AND stage_status.code <> 'release'
                              AND stage_status.ordering >=
                                  candidate.current_status_ordering
                              AND sla.planned_date IS NOT NULL
                              AND sla.planned_date <
                                  :#{#params.todayStart}
                        )
                    )

                    OR

                    /* STAGE_DEADLINES_NOT_FILLED */
                    (
                        :#{#params.stageDeadlinesNotFilledEnabled} = true
                        AND EXISTS (
                            SELECT 1
                            FROM status stage_status
                            LEFT JOIN agent_status_sla sla
                                ON sla.agent_status_id = stage_status.id
                               AND sla.ai_agent_id = candidate.id
                            WHERE stage_status.disabled = false
                              AND stage_status.code <> 'release'
                              AND stage_status.ordering >=
                                  candidate.current_status_ordering
                              AND (
                                  sla.ai_agent_id IS NULL
                                  OR (
                                      sla.completed_date IS NULL
                                      AND sla.planned_date IS NULL
                                  )
                              )
                        )
                    )

                    OR

                    /* GIGAUSAGE_NOT_FILLED */
                    (
                        :#{#params.gigaUsageNotFilledEnabled} = true

                        AND candidate.current_status_code IN (
                            :#{#params.pilotStatus},
                            :#{#params.targetSolutionStatus}
                        )

                        AND NOT EXISTS (
                            SELECT 1
                            FROM jira_issue jira
                            WHERE jira.agent_id = candidate.id
                              AND LOWER(jira.project) =
                                  LOWER(:#{#params.gigaUsageProject})
                              AND NULLIF(
                                  BTRIM(jira.jira_key),
                                  ''
                              ) IS NOT NULL
                        )
                    )

                    OR

                    /* ENABLERS_NOT_FILLED */
                    (
                        :#{#params.enablersNotFilledEnabled} = true
                        AND NOT EXISTS (
                            SELECT 1
                            FROM agent_enabler agent_enabler
                            WHERE agent_enabler.agent_id =
                                candidate.id
                        )
                    )

                    OR

                    /* STAGE_DEADLINE_TODAY */
                    (
                        :#{#params.stageDeadlineTodayEnabled} = true
                        AND EXISTS (
                            SELECT 1
                            FROM agent_status_sla sla
                            INNER JOIN status stage_status
                                ON stage_status.id = sla.agent_status_id
                            WHERE sla.ai_agent_id = candidate.id
                              AND sla.completed_date IS NULL
                              AND stage_status.disabled = false
                              AND stage_status.code <> 'release'
                              AND stage_status.ordering >=
                                  candidate.current_status_ordering
                              AND sla.planned_date >=
                                  :#{#params.todayStart}
                              AND sla.planned_date <
                                  :#{#params.tomorrowStart}
                        )
                    )

                    OR

                    /* STAGE_DEADLINE_TOMORROW */
                    (
                        :#{#params.stageDeadlineTomorrowEnabled} = true
                        AND EXISTS (
                            SELECT 1
                            FROM agent_status_sla sla
                            INNER JOIN status stage_status
                                ON stage_status.id = sla.agent_status_id
                            WHERE sla.ai_agent_id = candidate.id
                              AND sla.completed_date IS NULL
                              AND stage_status.disabled = false
                              AND stage_status.code <> 'release'
                              AND stage_status.ordering >=
                                  candidate.current_status_ordering
                              AND sla.planned_date >=
                                  :#{#params.tomorrowStart}
                              AND sla.planned_date <
                                  :#{#params.inTwoDaysStart}
                        )
                    )

                    OR

                    /* STAGE_DEADLINE_IN_2_DAYS */
                    (
                        :#{#params.stageDeadlineInTwoDaysEnabled} = true
                        AND EXISTS (
                            SELECT 1
                            FROM agent_status_sla sla
                            INNER JOIN status stage_status
                                ON stage_status.id = sla.agent_status_id
                            WHERE sla.ai_agent_id = candidate.id
                              AND sla.completed_date IS NULL
                              AND stage_status.disabled = false
                              AND stage_status.code <> 'release'
                              AND stage_status.ordering >=
                                  candidate.current_status_ordering
                              AND sla.planned_date >=
                                  :#{#params.inTwoDaysStart}
                              AND sla.planned_date <
                                  :#{#params.inThreeDaysStart}
                        )
                    )

                    OR

                    /* STAGE_DEADLINE_IN_3_DAYS */
                    (
                        :#{#params.stageDeadlineInThreeDaysEnabled} = true
                        AND EXISTS (
                            SELECT 1
                            FROM agent_status_sla sla
                            INNER JOIN status stage_status
                                ON stage_status.id = sla.agent_status_id
                            WHERE sla.ai_agent_id = candidate.id
                              AND sla.completed_date IS NULL
                              AND stage_status.disabled = false
                              AND stage_status.code <> 'release'
                              AND stage_status.ordering >=
                                  candidate.current_status_ordering
                              AND sla.planned_date >=
                                  :#{#params.inThreeDaysStart}
                              AND sla.planned_date <
                                  :#{#params.inFourDaysStart}
                        )
                    )

                    OR

                    /* AI_OBJECT_REQUIRED */
                    (
                        :#{#params.aiObjectRequiredEnabled} = true

                        AND candidate.current_status_code IN (
                            :#{#params.pilotStatus},
                            :#{#params.targetSolutionStatus}
                        )

                        AND EXISTS (
                            SELECT 1
                            FROM jira_issue jira
                            WHERE jira.agent_id = candidate.id
                              AND LOWER(jira.project) =
                                  LOWER(:#{#params.crossgoalProject})
                              AND LOWER(jira.type) = 'initiative'
                              AND NULLIF(
                                  BTRIM(jira.jira_key),
                                  ''
                              ) IS NOT NULL
                        )

                        AND NOT EXISTS (
                            SELECT 1
                            FROM initiative_meta_ai_object ai_object
                            WHERE ai_object.initiative_id = candidate.id
                              AND ai_object.is_active IS TRUE
                        )
                    )
            ),

            remaining_candidates AS (
                SELECT candidate.*
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

                  /* Такое же ограничение как в showcase */
                  AND candidate.current_status_code IN (
                      :#{#params.pilotStatus},
                      :#{#params.targetSolutionStatus}
                  )

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
                            FROM initiative_metric_assignment assignment
                            WHERE assignment.initiative_agent_type_id = metric_type.id
                              AND assignment.metric_id = metric.id
                              AND assignment.applicability_status <> 'ACTIVE'
                        )

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
        """
    )
    fun countInitiativesWithDeviations(
        @Param("params") params: InitiativeDeviationCountQueryParams
    ): Long
}
    
```
