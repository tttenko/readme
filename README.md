```java

@Service
class InitiativeMetricValueReader(

    private val messageProvider: MessageProvider,
    private val initiativeMetricTypeRepository: InitiativeMetricTypeRepository,
    private val initiativeMetricValueRepository: InitiativeMetricValueRepository,
    private val metricsDirectoryRepository: MetricsDirectoryRepository,
    private val metricResponseBuilder: InitiativeMetricResponseBuilder,
) {

    @Transactional(readOnly = true)
    fun getInitiativeMetricValues(initiativeId: Long): List<InitiativeMetricResponse> {
        val metricTypes =
            initiativeMetricTypeRepository.findAllByAiAgentId(
                initiativeId = initiativeId
            )

        if (metricTypes.isEmpty()) {
            return emptyList()
        }

        val requestedAgentTypes =
            metricTypes
                .map { metricType ->
                    InitiativeMetricAgentType.fromValue(metricType.agentType.orEmpty())
                        ?: throw AiBadRequestException(
                            errorCode = WRONG_INITIATIVE_METRIC_AGENT_TYPE,
                            message = MessageFormat.format(
                                messageProvider[WRONG_INITIATIVE_METRIC_AGENT_TYPE],
                                metricType.agentType,
                            ),
                        )
                }
                .toSet()

        val metrics =
            metricsDirectoryRepository.findApplicableMetrics(
                autonomousSelected =
                    requestedAgentTypes.contains(InitiativeMetricAgentType.AUTONOMOUS),
                copilotSelected =
                    requestedAgentTypes.contains(InitiativeMetricAgentType.COPILOT),
                appealsSelected =
                    requestedAgentTypes.contains(InitiativeMetricAgentType.APPEALS),
            )

        if (metrics.isEmpty()) {
            return emptyList()
        }

        val reportingMonth = YearMonth.now().minusMonths(1)
        val previousPeriodMonth = reportingMonth.minusMonths(1)

        val hasReportingMonthValues =
            initiativeMetricValueRepository
                .existsByInitiativeMetricTypeAiAgentIdAndPeriodMonth(
                    initiativeId = initiativeId,
                    periodMonth = reportingMonth.atDay(1),
                )

        val metricValues =
            initiativeMetricValueRepository
                .findValuesForInitiativeMetricsInPeriodRange(
                    initiativeId = initiativeId,
                    agentTypes = requestedAgentTypes
                        .map { it.value }
                        .toSet(),
                    metricDirectoryIds = metrics
                        .map { it.id }
                        .toSet(),
                    periodFrom = previousPeriodMonth.atDay(1),
                    periodTo = reportingMonth.atDay(1),
                )

        val metricIdsWithSubmittedValue =
            metricValues
                .asSequence()
                .filter { metricValue ->
                    metricValue.metricValue != null ||
                        metricValue.targetValue != null
                }
                .mapNotNull { metricValue ->
                    metricValue.metricDirectory?.id
                }
                .toSet()

        return metricResponseBuilder
            .build(
                metrics = metrics,
                requestedAgentTypes = requestedAgentTypes,
                metricValues = metricValues,
                reportingMonth = reportingMonth,
                clearRegularMetricValue = !hasReportingMonthValues,
            )
            .filter { response ->
                response.isActive != false ||
                    response.id in metricIdsWithSubmittedValue
            }
            .sortedBy { response ->
                response.isActive == false
            }
    }
}

@Component
class InitiativeMetricResponseBuilder(
    private val metricApplicabilityPolicy: InitiativeMetricApplicabilityPolicy,
    private val planExecutionCalculator: InitiativeMetricPlanExecutionCalculator,
) {

    fun buildWithoutValues(
        metrics: List<MetricsDirectoryEntity>,
        requestedAgentTypes: Set<InitiativeMetricAgentType>,
    ): List<InitiativeMetricResponse> {

        return metrics.flatMap { metric ->
            val availableAgentTypesForMetric =
                metricApplicabilityPolicy.findApplicableAgentTypes(
                    metric = metric,
                    requestedAgentTypes = requestedAgentTypes,
                )

            availableAgentTypesForMetric.map { agentType ->
                InitiativeMetricResponse(
                    id = metric.id,
                    name = metric.name,
                    unit = metric.unit,
                    direction = metric.direction,
                    agentType = agentType.value,
                    isActive = metric.active,
                    description = metric.description,
                    frequency = metric.frequency,
                    metricValue = null,
                    targetValue = null,
                    planExecution = null,
                    periods = emptyList(),
                )
            }
        }
    }

    fun build(
        metrics: List<MetricsDirectoryEntity>,
        requestedAgentTypes: Set<InitiativeMetricAgentType>,
        metricValues: List<InitiativeMetricValueEntity>,
        reportingMonth: YearMonth,
        clearRegularMetricValue: Boolean = false,
    ): List<InitiativeMetricResponse> {

        val metricValuesByMetricAndAgentType =
            metricValues.groupBy { metricValue ->
                MetricValueKey(
                    metricId = metricValue.metricDirectory?.id,
                    agentType = metricValue.initiativeMetricType?.agentType,
                )
            }

        return metrics.flatMap { metric ->
            val metricId = metric.id

            val availableAgentTypesForMetric =
                metricApplicabilityPolicy.findApplicableAgentTypes(
                    metric = metric,
                    requestedAgentTypes = requestedAgentTypes,
                )

            availableAgentTypesForMetric.map { agentType ->

                val valuesForMetric =
                    metricValuesByMetricAndAgentType[
                        MetricValueKey(
                            metricId = metricId,
                            agentType = agentType.value,
                        )
                    ].orEmpty()

                val latestMetricValue =
                    valuesForMetric.maxByOrNull { metricValue ->
                        metricValue.periodMonth ?: LocalDate.MIN
                    }

                val metricValue =
                    latestMetricValue
                        ?.metricValue
                        ?.takeUnless {
                            clearRegularMetricValue &&
                                metric.frequency.equals(
                                    REGULAR_FREQUENCY,
                                    ignoreCase = true,
                                )
                        }

                val targetValue =
                    latestMetricValue
                        ?.targetValue
                        ?.takeUnless {
                            metric.frequency.equals(
                                CONSTANT_FREQUENCY,
                                ignoreCase = true,
                            )
                        }

                InitiativeMetricResponse(
                    id = metricId,
                    name = metric.name,
                    unit = metric.unit,
                    direction = metric.direction,
                    agentType = agentType.value,
                    isActive = metric.active,
                    description = metric.description,
                    frequency = metric.frequency,
                    metricValue = metricValue,
                    targetValue = targetValue,
                    planExecution =
                        planExecutionCalculator.calculate(
                            direction = metric.direction,
                            metricValue = metricValue,
                            targetValue = targetValue,
                        ),
                    periods = buildPreviousPeriod(
                        metricValues = valuesForMetric,
                        reportingMonth = reportingMonth,
                    ),
                )
            }
        }
    }

    private fun buildPreviousPeriod(
        metricValues: List<InitiativeMetricValueEntity>,
        reportingMonth: YearMonth,
    ): List<InitiativeMetricPeriodResponse> {

        val previousPeriodMonth = reportingMonth.minusMonths(1)

        val previousPeriodValue =
            metricValues.find { metricValue ->
                metricValue.periodMonth == previousPeriodMonth.atDay(1)
            } ?: return emptyList()

        return listOf(
            InitiativeMetricPeriodResponse(
                index = PREVIOUS_PERIOD_INDEX,
                period = previousPeriodMonth
                    .atDay(1)
                    .atStartOfDay(ZoneOffset.UTC)
                    .toInstant(),
                value = previousPeriodValue.metricValue,
            )
        )
    }

    private data class MetricValueKey(
        val metricId: UUID?,
        val agentType: String?,
    )

    companion object {
        private const val PREVIOUS_PERIOD_INDEX = 1
        private const val REGULAR_FREQUENCY = "regular"
        private const val CONSTANT_FREQUENCY = "constant"
    }
}
```
