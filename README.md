```java

private fun hasUnlinkedMetric(
    requests: List<SaveInitiativeMetricValueRequest>,
    initiativeMetricTypesMap: Map<String, InitiativeMetricTypeEntity>,
    assignments: List<InitiativeMetricAssignmentEntity>,
    periodMonth: LocalDate,
): Boolean {
    if (assignments.isEmpty()) return false

    val assignmentsMap = assignments.associateBy { assignment ->
        assignment.initiativeMetricType.id to assignment.metric.id
    }

    val unlinkedAssignmentIds = metricApplicabilityRequestRepository
        .findUnlinkedAssignmentIdsForPeriod(
            assignmentIds = assignments.map { it.id }.toSet(),
            periodMonth = periodMonth
        )
        .toSet()

    return requests.any { metricValueRequest ->
        val agentType = validateInitiativeMetricAgentType(metricValueRequest.agentType.trim())
        val initiativeMetricType = initiativeMetricTypesMap.getValue(agentType.value)
        val assignment = assignmentsMap[initiativeMetricType.id to metricValueRequest.metricId]

        assignment != null && assignment.id in unlinkedAssignmentIds
    }
}

@Query(
    """
    select distinct r.initiativeMetricAssignment.id
    from MetricApplicabilityRequestEntity r
    where r.initiativeMetricAssignment.id in :assignmentIds
      and r.status in :statuses
      and r.effectiveFromPeriod <= :periodMonth
      and (r.effectiveToPeriod is null or :periodMonth < r.effectiveToPeriod)
    """
)
fun findUnlinkedAssignmentIdsForPeriod(
    @Param("assignmentIds") assignmentIds: Set<Long>,
    @Param("periodMonth") periodMonth: LocalDate,
    @Param("statuses") statuses: Set<MetricApplicabilityRequestStatus> = setOf(
        MetricApplicabilityRequestStatus.PENDING,
        MetricApplicabilityRequestStatus.APPROVED
    ),
): List<Long>
```
