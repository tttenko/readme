```java

@Component
class GigaUsageNotFilledStrategy : InitiativeDeviationStrategy {

    override val code = InitiativeDeviationCode.GIGAUSAGE_NOT_FILLED

    override val evaluationOrder: Int = 30

    override fun findMatchingInitiativeIds(
        candidateInitiativeIds: Set<Long>,
        session: InitiativeDeviationCalculationSession,
        context: InitiativeDeviationEvaluationContext,
    ): Set<Long> =
        candidateInitiativeIds -
            session.initiativeIdsWithValidGigaUsage
}

@Query(
    """
        select distinct jiraIssue.agent.id
        from JiraIssueEntity jiraIssue
        where jiraIssue.agent.id in :initiativeIds
          and lower(jiraIssue.project) = 'gigausage'
          and jiraIssue.jiraKey is not null
          and trim(jiraIssue.jiraKey) <> ''
    """
)
fun findInitiativeIdsWithValidGigaUsage(
    @Param("initiativeIds")
    initiativeIds: Collection<Long>,
): Set<Long>


@Component
class EnablersNotFilledStrategy : InitiativeDeviationStrategy {

    override val code = InitiativeDeviationCode.ENABLERS_NOT_FILLED

    override val evaluationOrder: Int = 40

    override fun findMatchingInitiativeIds(
        candidateInitiativeIds: Set<Long>,
        session: InitiativeDeviationCalculationSession,
        context: InitiativeDeviationEvaluationContext,
    ): Set<Long> =
        candidateInitiativeIds -
            session.initiativeIdsWithEnablers
}

@Query(
    value = """
        select distinct agent_enabler.agent_id
        from agent_enabler
        where agent_enabler.agent_id in (:initiativeIds)
    """,
    nativeQuery = true,
)
fun findInitiativeIdsWithEnablers(
    @Param("initiativeIds")
    initiativeIds: Collection<Long>,
): Set<Long>




```
