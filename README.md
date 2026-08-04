```java
abstract class AbstractStageDeadlineStrategy(
    final override val code: InitiativeDeviationCode,
    final override val evaluationOrder: Int,
) : InitiativeDeviationStrategy {

    protected fun unfinishedStages(
        initiativeId: Long,
        session: InitiativeDeviationCalculationSession,
    ): List<AgentStatusSlaEntity> {
        val currentStatusOrdering =
            session.statusOrderingByInitiativeId[initiativeId]
                ?: return emptyList()

        return session.statusSlaByInitiativeId[initiativeId]
            .orEmpty()
            .filter { statusSla ->
                statusSla.completedDate == null
            }
            .filter { statusSla ->
                val statusId =
                    statusSla.primaryKey.agentStatusId
                        ?: return@filter false

                val stageOrdering =
                    session.activeStatusOrderingById[statusId]
                        ?: return@filter false

                stageOrdering >= currentStatusOrdering
            }
    }
}
  ```
