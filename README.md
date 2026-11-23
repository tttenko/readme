```java

abstract class AbstractStageDeadlineStrategy(
    final override val code: InitiativeDeviationCode,
    final override val evaluationOrder: Int,
) : InitiativeDeviationStrategy {

    protected fun relevantStageIds(
        initiativeId: Long,
        session: InitiativeDeviationCalculationSession,
    ): Set<Long> {
        val currentStatusOrdering = session.statusOrderingByInitiativeId[initiativeId] ?: return emptySet()

        return session.activeStatuses
            .asSequence()
            .filter { status ->
                status.code != RELEASE_STATUS_CODE &&
                    status.ordering?.let { it >= currentStatusOrdering } == true
            }
            .map { it.id }
            .toSet()
    }

    protected fun unfinishedStages(
        initiativeId: Long,
        session: InitiativeDeviationCalculationSession,
    ): List<AgentStatusSlaEntity> {
        val relevantStatusIds = relevantStageIds(initiativeId, session)

        return session.statusSlaByInitiativeId[initiativeId]
            .orEmpty()
            .filter { statusSla ->
                statusSla.completedDate == null &&
                    statusSla.primaryKey.agentStatusId in relevantStatusIds
            }
    }

    private companion object {
        const val RELEASE_STATUS_CODE = "release"
    }
}

```
