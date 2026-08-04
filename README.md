```java
class InitiativeDeviationCalculationSession(
    val initiativeIds: Set<Long>,
    private val agentStatusSlaRepository: AgentStatusSlaRepository,
    private val jiraIssueRepository: JiraIssueRepository,
    private val enablerRepository: EnablerRepository,
    private val aiAgentRepository: AIAgentRepository,
    private val statusRepository: StatusRepository,
    val currentPeriodMetricsDeviationFinder: CurrentPeriodMetricsDeviationFinder,
) {

    val statusSlaByInitiativeId: Map<Long, List<AgentStatusSlaEntity>> by lazy {
        if (initiativeIds.isEmpty()) {
            emptyMap()
        } else {
            agentStatusSlaRepository
                .findAllByAiAgentIdIn(initiativeIds)
                .mapNotNull { statusSla ->
                    statusSla.primaryKey.aiAgentId
                        ?.let { initiativeId -> initiativeId to statusSla }
                }
                .groupBy(
                    keySelector = { it.first },
                    valueTransform = { it.second },
                )
        }
    }

    private val initiativeStatuses: List<InitiativeStatusProjection> by lazy {
        if (initiativeIds.isEmpty()) {
            emptyList()
        } else {
            aiAgentRepository.findInitiativeStatuses(initiativeIds)
        }
    }

    val statusCodeByInitiativeId: Map<Long, String?> by lazy {
        initiativeStatuses.associate { projection ->
            projection.initiativeId to projection.statusCode
        }
    }

    val statusOrderingByInitiativeId: Map<Long, Long?> by lazy {
        initiativeStatuses.associate { projection ->
            projection.initiativeId to projection.statusOrdering
        }
    }

    val activeStatuses: List<StatusEntity> by lazy {
        statusRepository.findAllByDisabledIsFalse()
    }

    val initiativeIdsWithValidGigaUsage: Set<Long> by lazy {
        if (initiativeIds.isEmpty()) {
            emptySet()
        } else {
            jiraIssueRepository
                .findInitiativeIdsWithValidGigaUsage(initiativeIds)
                .toSet()
        }
    }

    val initiativeIdsWithEnablers: Set<Long> by lazy {
        if (initiativeIds.isEmpty()) {
            emptySet()
        } else {
            enablerRepository
                .findInitiativeIdsWithEnablers(initiativeIds)
                .toSet()
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
            val currentStatusOrdering =
                session.statusOrderingByInitiativeId[initiativeId]
                    ?: return@filterTo false

            val currentAndFutureStatusIds =
                session.activeStatuses
                    .asSequence()
                    .filter { status ->
                        status.ordering != null &&
                            status.ordering!! >= currentStatusOrdering
                    }
                    .map { status -> status.id }
                    .toSet()

            val statusSlaByStatusId =
                session.statusSlaByInitiativeId[initiativeId]
                    .orEmpty()
                    .mapNotNull { statusSla ->
                        statusSla.primaryKey.agentStatusId
                            ?.let { statusId -> statusId to statusSla }
                    }
                    .toMap()

            currentAndFutureStatusIds.any { statusId ->
                val statusSla = statusSlaByStatusId[statusId]

                statusSla == null ||
                    (
                        statusSla.completedDate == null &&
                            statusSla.plannedDate == null
                    )
            }
        }
}
  ```
