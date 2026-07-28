```java

@Component
class CurrentPeriodMetricsDeviationFinder(
    private val initiativeMetricTypeRepository: InitiativeMetricTypeRepository,
    private val metricsDirectoryRepository: MetricsDirectoryRepository,
    private val initiativeMetricValueRepository: InitiativeMetricValueRepository,
) {

    fun findInitiativeIdsWithMissingMetrics(
        initiativeIds: Set<Long>,
        currentPeriod: LocalDate,
    ): Set<Long> {
        if (initiativeIds.isEmpty()) {
            return emptySet()
        }

        val metricTypes =
            initiativeMetricTypeRepository.findAllByAiAgentIdIn(initiativeIds)

        val metricTypesByInitiativeId =
            metricTypes
                .filter { metricType -> metricType.aiAgent?.id != null }
                .groupBy { metricType -> requireNotNull(metricType.aiAgent?.id) }

        val activeMetrics =
            metricsDirectoryRepository.findAllByActiveIsTrue()

        val metricTypeIds =
            metricTypes.mapTo(mutableSetOf()) { metricType -> metricType.id }

        val filledKeys =
            if (metricTypeIds.isEmpty()) {
                emptySet()
            } else {
                initiativeMetricValueRepository
                    .findAllByInitiativeMetricTypeIdsAndPeriodMonth(
                        initiativeMetricTypeIds = metricTypeIds,
                        periodMonth = currentPeriod,
                    )
                    .asSequence()
                    .filter { metricValue -> metricValue.metricValue != null }
                    .mapNotNull { metricValue ->
                        val metricTypeId =
                            metricValue.initiativeMetricType?.id
                                ?: return@mapNotNull null

                        val metricId =
                            metricValue.metricDirectory?.id
                                ?: return@mapNotNull null

                        MetricCoverageKey(
                            initiativeMetricTypeId = metricTypeId,
                            metricId = metricId,
                        )
                    }
                    .toSet()
            }

        return initiativeIds.filterTo(mutableSetOf()) { initiativeId ->
            val initiativeMetricTypes =
                metricTypesByInitiativeId[initiativeId].orEmpty()

            val expectedKeys =
                initiativeMetricTypes.flatMapTo(mutableSetOf()) { metricType ->
                    activeMetrics
                        .asSequence()
                        .filter { metric ->
                            metric.isApplicableTo(metricType.agentType)
                        }
                        .map { metric ->
                            MetricCoverageKey(
                                initiativeMetricTypeId = metricType.id,
                                metricId = metric.id,
                            )
                        }
                        .toList()
                }

            expectedKeys.isNotEmpty() &&
                !filledKeys.containsAll(expectedKeys)
        }
    }

    private fun MetricsDirectoryEntity.isApplicableTo(
        rawAgentType: String?,
    ): Boolean =
        when (InitiativeMetricAgentType.fromValue(rawAgentType.orEmpty())) {
            InitiativeMetricAgentType.AUTONOMOUS ->
                autonomousApplicability == true

            InitiativeMetricAgentType.COPILOT ->
                copilotApplicability == true

            InitiativeMetricAgentType.APPEALS ->
                requiresAppealsWork == true

            null -> false
        }

    private data class MetricCoverageKey(
        val initiativeMetricTypeId: Long,
        val metricId: UUID,
    )
}

```
