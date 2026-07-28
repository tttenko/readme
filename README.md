```java

class InitiativeDeviationCalculationSession(
    val initiativeIds: Set<Long>,
    private val agentStatusSlaRepository: AgentStatusSlaRepository,
    private val jiraIssueRepository: JiraIssueRepository,
    private val enablerRepository: EnablerRepository,
    private val aiAgentRepository: AIAgentRepository,
    val currentPeriodMetricsDeviationFinder: CurrentPeriodMetricsDeviationFinder,
) {

    val statusSlaByInitiativeId: Map<Long, List<AgentStatusSlaEntity>> by lazy {
        if (initiativeIds.isEmpty()) {
            emptyMap()
        } else {
            agentStatusSlaRepository
                .findAllByAiAgentIdIn(initiativeIds)
                .filter { it.aiAgent?.id != null }
                .groupBy { requireNotNull(it.aiAgent?.id) }
        }
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

    val statusCodeByInitiativeId: Map<Long, String?> by lazy {
        if (initiativeIds.isEmpty()) {
            emptyMap()
        } else {
            aiAgentRepository
                .findInitiativeStatuses(initiativeIds)
                .associate { projection ->
                    projection.initiativeId to projection.statusCode
                }
        }
    }
}

```
