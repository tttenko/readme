```java

@Component
class CurrentPeriodMetricsDeviationFinder(
    private val initiativeMetricTypeRepository: InitiativeMetricTypeRepository,
    private val metricsDirectoryRepository: MetricsDirectoryRepository,
    private val initiativeMetricValueRepository: InitiativeMetricValueRepository,
    private val metricApplicabilityPolicy: InitiativeMetricApplicabilityPolicy,
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

        if (metricTypes.isEmpty()) {
            return emptySet()
        }

        val metricTypesByInitiativeId =
            metricTypes
                .filter { metricType -> metricType.aiAgent?.id != null }
                .groupBy { metricType ->
                    requireNotNull(metricType.aiAgent?.id)
                }

        val activeMetrics =
            metricsDirectoryRepository.findAllByActiveIsTrue()

        if (activeMetrics.isEmpty()) {
            return emptySet()
        }

        val metricTypeIds =
            metricTypes.mapTo(mutableSetOf()) { metricType ->
                metricType.id
            }

        val filledKeys =
            initiativeMetricValueRepository
                .findAllByInitiativeMetricTypeIdsAndPeriodMonth(
                    initiativeMetricTypeIds = metricTypeIds,
                    periodMonth = currentPeriod,
                )
                .asSequence()
                .filter { metricValue ->
                    metricValue.metricValue != null
                }
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

        return initiativeIds.filterTo(mutableSetOf()) { initiativeId ->
            val initiativeMetricTypes =
                metricTypesByInitiativeId[initiativeId].orEmpty()

            val expectedKeys =
                initiativeMetricTypes.flatMapTo(mutableSetOf()) { metricType ->
                    val agentType =
                        InitiativeMetricAgentType.fromValue(
                            metricType.agentType.orEmpty(),
                        ) ?: return@flatMapTo emptyList()

                    activeMetrics
                        .filter { metric ->
                            metricApplicabilityPolicy.isApplicable(
                                metric = metric,
                                agentType = agentType,
                            )
                        }
                        .map { metric ->
                            MetricCoverageKey(
                                initiativeMetricTypeId = metricType.id,
                                metricId = metric.id,
                            )
                        }
                }

            expectedKeys.isNotEmpty() &&
                !filledKeys.containsAll(expectedKeys)
        }
    }

    private data class MetricCoverageKey(
        val initiativeMetricTypeId: Long,
        val metricId: UUID,
    )
}
```
