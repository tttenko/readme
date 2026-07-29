```java

interface InitiativeStatusProjection {
    val initiativeId: Long
    val statusCode: String?
}

@Query(
    """
        select
            agent.id as initiativeId,
            status.code as statusCode
        from AIAgentEntity agent
        left join agent.agentStatus status
        where agent.id in :initiativeIds
    """
)
fun findInitiativeStatuses(
    @Param("initiativeIds")
    initiativeIds: Collection<Long>,
): List<InitiativeStatusProjection>

fun findAllByAiAgentIdIn(
    initiativeIds: Collection<Long>,
): List<InitiativeMetricTypeEntity>

fun findAllByActiveIsTrue(): List<MetricsDirectoryEntity>

@Query(
    """
        select metricValue
        from InitiativeMetricValueEntity metricValue
        join fetch metricValue.initiativeMetricType initiativeMetricType
        join fetch metricValue.metricDirectory metricDirectory
        where initiativeMetricType.id in :initiativeMetricTypeIds
          and metricValue.periodMonth = :periodMonth
    """
)
fun findAllByInitiativeMetricTypeIdsAndPeriodMonth(
    @Param("initiativeMetricTypeIds")
    initiativeMetricTypeIds: Collection<Long>,

    @Param("periodMonth")
    periodMonth: LocalDate,
): List<InitiativeMetricValueEntity>





```
