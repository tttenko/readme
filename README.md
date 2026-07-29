```java

@Component
class InitiativeDeviationCalculationSessionFactory(
    private val agentStatusSlaRepository: AgentStatusSlaRepository,
    private val jiraIssueRepository: JiraIssueRepository,
    private val enablerRepository: EnablerRepository,
    private val aiAgentRepository: AIAgentRepository,
    private val currentPeriodMetricsDeviationFinder: CurrentPeriodMetricsDeviationFinder,
) {

    fun create(
        initiativeIds: Set<Long>,
    ): InitiativeDeviationCalculationSession =
        InitiativeDeviationCalculationSession(
            initiativeIds = initiativeIds,
            agentStatusSlaRepository = agentStatusSlaRepository,
            jiraIssueRepository = jiraIssueRepository,
            enablerRepository = enablerRepository,
            aiAgentRepository = aiAgentRepository,
            currentPeriodMetricsDeviationFinder = currentPeriodMetricsDeviationFinder,
        )
}

abstract class AbstractStageDeadlineStrategy(
    final override val code: InitiativeDeviationCode,
    final override val evaluationOrder: Int,
) : InitiativeDeviationStrategy {

    protected fun unfinishedStages(
        initiativeId: Long,
        session: InitiativeDeviationCalculationSession,
    ): List<AgentStatusSlaEntity> =
        session.statusSlaByInitiativeId[initiativeId]
            .orEmpty()
            .filter { statusSla -> statusSla.completedDate == null }
}

@Component
class StageDeadlineExpiredStrategy :
    AbstractStageDeadlineStrategy(
        code = InitiativeDeviationCode.STAGE_DEADLINE_EXPIRED,
        evaluationOrder = 10,
    ) {

    override fun findMatchingInitiativeIds(
        candidateInitiativeIds: Set<Long>,
        session: InitiativeDeviationCalculationSession,
        context: InitiativeDeviationEvaluationContext,
    ): Set<Long> =
        candidateInitiativeIds.filterTo(mutableSetOf()) { initiativeId ->
            unfinishedStages(initiativeId, session)
                .any { statusSla ->
                    statusSla.plannedDate
                        ?.toLocalDate()
                        ?.isBefore(context.today) == true
                }
        }
}

@Component
class StageDeadlinesNotFilledStrategy :
    AbstractStageDeadlineStrategy(
        code = InitiativeDeviationCode.STAGE_DEADLINES_NOT_FILLED,
        evaluationOrder = 20,
    ) {

    override fun findMatchingInitiativeIds(
        candidateInitiativeIds: Set<Long>,
        session: InitiativeDeviationCalculationSession,
        context: InitiativeDeviationEvaluationContext,
    ): Set<Long> =
        candidateInitiativeIds.filterTo(mutableSetOf()) { initiativeId ->
            unfinishedStages(initiativeId, session)
                .any { statusSla -> statusSla.plannedDate == null }
        }
}

abstract class AbstractUpcomingStageDeadlineStrategy(
    code: InitiativeDeviationCode,
    evaluationOrder: Int,
    private val daysBeforeDeadline: Long,
) : AbstractStageDeadlineStrategy(
    code = code,
    evaluationOrder = evaluationOrder,
) {

    override fun findMatchingInitiativeIds(
        candidateInitiativeIds: Set<Long>,
        session: InitiativeDeviationCalculationSession,
        context: InitiativeDeviationEvaluationContext,
    ): Set<Long> {
        val expectedDate = context.today.plusDays(daysBeforeDeadline)

        return candidateInitiativeIds.filterTo(mutableSetOf()) { initiativeId ->
            unfinishedStages(initiativeId, session)
                .any { statusSla ->
                    statusSla.plannedDate?.toLocalDate() == expectedDate
                }
        }
    }
}

abstract class AbstractUpcomingStageDeadlineStrategy(
    code: InitiativeDeviationCode,
    evaluationOrder: Int,
    private val daysBeforeDeadline: Long,
) : AbstractStageDeadlineStrategy(
    code = code,
    evaluationOrder = evaluationOrder,
) {

    override fun findMatchingInitiativeIds(
        candidateInitiativeIds: Set<Long>,
        session: InitiativeDeviationCalculationSession,
        context: InitiativeDeviationEvaluationContext,
    ): Set<Long> {
        val expectedDate = context.today.plusDays(daysBeforeDeadline)

        return candidateInitiativeIds.filterTo(mutableSetOf()) { initiativeId ->
            unfinishedStages(initiativeId, session)
                .any { statusSla ->
                    statusSla.plannedDate?.toLocalDate() == expectedDate
                }
        }
    }
}

@Component
class StageDeadlineTomorrowStrategy :
    AbstractUpcomingStageDeadlineStrategy(
        code = InitiativeDeviationCode.STAGE_DEADLINE_TOMORROW,
        evaluationOrder = 50,
        daysBeforeDeadline = 1,
    )

@Component
class StageDeadlineInTwoDaysStrategy :
    AbstractUpcomingStageDeadlineStrategy(
        code = InitiativeDeviationCode.STAGE_DEADLINE_IN_2_DAYS,
        evaluationOrder = 60,
        daysBeforeDeadline = 2,
    )

@Component
class StageDeadlineInThreeDaysStrategy :
    AbstractUpcomingStageDeadlineStrategy(
        code = InitiativeDeviationCode.STAGE_DEADLINE_IN_3_DAYS,
        evaluationOrder = 70,
        daysBeforeDeadline = 3,
    )



```
