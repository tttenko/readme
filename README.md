```java
@Service
class InitiativeMetricPreAnalyticsReader(
    private val messageProvider: MessageProvider,
    private val preAnalyticsProperties: PreAnalyticsProperties,
    private val initiativeMetricTypeRepository:
        InitiativeMetricTypeRepository,
    private val initiativeMetricValueRepository:
        InitiativeMetricValueRepository,
    private val metricsDirectoryRepository:
        MetricsDirectoryRepository,
    private val responseBuilder:
        InitiativeMetricPreAnalyticsResponseBuilder,
) {

    @Transactional(readOnly = true)
    fun getPreAnalytics(
        initiativeId: Long,
        now: Instant = Instant.now(),
    ): InitiativeMetricPreAnalyticsResponse {
        val metricTypes =
            initiativeMetricTypeRepository
                .findAllByAiAgentIdAndAgentTypeIn(
                    initiativeId = initiativeId,
                    agentTypes = SUPPORTED_AGENT_TYPES,
                )

        if (metricTypes.isEmpty()) {
            throw AiConflictException(
                errorCode =
                    INITIATIVE_METRIC_TYPES_NOT_FOUND,
                message = MessageFormat.format(
                    messageProvider[
                        INITIATIVE_METRIC_TYPES_NOT_FOUND
                    ],
                    initiativeId,
                ),
            )
        }

        val configuredMetricNames =
            preAnalyticsProperties.metricNames

        check(
            configuredMetricNames.size ==
                EXPECTED_METRICS_COUNT &&
                configuredMetricNames.toSet().size ==
                EXPECTED_METRICS_COUNT
        ) {
            "Exactly $EXPECTED_METRICS_COUNT unique " +
                "pre-analytics metric names must be configured"
        }

        val metricsByName =
            metricsDirectoryRepository
                .findAllByNameIn(configuredMetricNames)
                .groupBy { metric -> metric.name }

        val missingMetricNames =
            configuredMetricNames.filterNot(
                metricsByName::containsKey
            )

        val duplicateMetricNames =
            metricsByName
                .filterValues { metrics ->
                    metrics.size > 1
                }
                .keys

        check(
            missingMetricNames.isEmpty() &&
                duplicateMetricNames.isEmpty()
        ) {
            "Invalid pre-analytics metrics configuration: " +
                "missing=$missingMetricNames, " +
                "duplicated=$duplicateMetricNames"
        }

        val metrics =
            configuredMetricNames.map { metricName ->
                metricsByName
                    .getValue(metricName)
                    .single()
            }

        val configuredMetricIds =
            metrics
                .map { metric -> metric.id }
                .toSet()

        val currentMonth =
            YearMonth.from(
                now.atZone(ZoneOffset.UTC)
            )

        val previousMonth =
            currentMonth.minusMonths(1)

        val beforePreviousMonth =
            currentMonth.minusMonths(2)

        val metricValues =
            initiativeMetricValueRepository
                .findValuesForInitiativeMetrics(
                    initiativeId = initiativeId,
                    agentTypes = SUPPORTED_AGENT_TYPES,
                    metricDirectoryIds =
                        configuredMetricIds,
                    periodFrom =
                        beforePreviousMonth.atDay(1),
                )

        val valuesByKey =
            metricValues
                .filter { metricValue ->
                    metricValue.periodMonth ==
                        previousMonth.atDay(1) ||
                        metricValue.periodMonth ==
                        beforePreviousMonth.atDay(1)
                }
                .associateBy { metricValue ->
                    InitiativeMetricPreAnalyticsResponseBuilder
                        .MetricValueKey(
                            agentType =
                                metricValue
                                    .initiativeMetricType
                                    ?.agentType
                                    .orEmpty(),
                            metricId = requireNotNull(
                                metricValue
                                    .metricDirectory
                                    ?.id
                            ),
                            periodMonth =
                                YearMonth.from(
                                    requireNotNull(
                                        metricValue.periodMonth
                                    )
                                ),
                        )
                }

        val metricsAutonomous =
            responseBuilder.buildMetricsForAgentType(
                agentType = AUTONOMOUS.value,
                metricTypes = metricTypes,
                metrics = metrics,
                valuesByKey = valuesByKey,
                previousMonth = previousMonth,
                beforePreviousMonth =
                    beforePreviousMonth,
            )

        val metricsCopilot =
            responseBuilder.buildMetricsForAgentType(
                agentType = COPILOT.value,
                metricTypes = metricTypes,
                metrics = metrics,
                valuesByKey = valuesByKey,
                previousMonth = previousMonth,
                beforePreviousMonth =
                    beforePreviousMonth,
            )

        val hasSubmittedValues =
            (metricsAutonomous + metricsCopilot)
                .any { metric ->
                    metric.value != null
                }

        return InitiativeMetricPreAnalyticsResponse(
            initiativeId = initiativeId,
            errorCode =
                METRIC_VALUES_NOT_SUBMITTED
                    .takeUnless {
                        hasSubmittedValues
                    },
            periodDisplayText =
                PERIOD_DISPLAY_TEXT_PREFIX +
                    currentMonth
                        .atDay(1)
                        .format(
                            PERIOD_DISPLAY_DATE_FORMATTER
                        ),
            metricsAutonomous = metricsAutonomous,
            metricsCopilot = metricsCopilot,
        )
    }

    private companion object {

        const val EXPECTED_METRICS_COUNT = 4

        const val METRIC_VALUES_NOT_SUBMITTED =
            "Metric values have not been submitted"

        const val PERIOD_DISPLAY_TEXT_PREFIX =
            "Значения на "

        val PERIOD_DISPLAY_DATE_FORMATTER:
            DateTimeFormatter =
            DateTimeFormatter.ofPattern(
                "dd.MM.yyyy"
            )

        val SUPPORTED_AGENT_TYPES =
            setOf(
                AUTONOMOUS.value,
                COPILOT.value,
            )
    }
}

@Component
class InitiativeMetricPreAnalyticsResponseBuilder {

    fun buildMetricsForAgentType(
        agentType: String,
        metricTypes:
            List<InitiativeMetricTypeEntity>,
        metrics:
            List<MetricsDirectoryEntity>,
        valuesByKey:
            Map<
                MetricValueKey,
                InitiativeMetricValueEntity
            >,
        previousMonth: YearMonth,
        beforePreviousMonth: YearMonth,
    ): List<InitiativeMetricPreAnalyticsItemResponse> {
        val agentTypeExists =
            metricTypes.any { metricType ->
                metricType.agentType == agentType
            }

        if (!agentTypeExists) {
            return emptyList()
        }

        return metrics.map { metric ->
            val previousValue =
                valuesByKey[
                    MetricValueKey(
                        agentType = agentType,
                        metricId = metric.id,
                        periodMonth = previousMonth,
                    )
                ]

            val beforePreviousValue =
                valuesByKey[
                    MetricValueKey(
                        agentType = agentType,
                        metricId = metric.id,
                        periodMonth =
                            beforePreviousMonth,
                    )
                ]

            buildMetricResponse(
                metric = metric,
                previousValue = previousValue,
                beforePreviousValue =
                    beforePreviousValue,
            )
        }
    }

    private fun buildMetricResponse(
        metric: MetricsDirectoryEntity,
        previousValue:
            InitiativeMetricValueEntity?,
        beforePreviousValue:
            InitiativeMetricValueEntity?,
    ): InitiativeMetricPreAnalyticsItemResponse {
        val submittedMetricValue =
            previousValue?.metricValue

        if (
            previousValue == null ||
            submittedMetricValue == null
        ) {
            return emptyMetricResponse(metric)
        }

        val comparisonValue =
            beforePreviousValue
                ?.metricValue
                ?.takeIf { value ->
                    submittedMetricValue.compareTo(
                        BigDecimal.ZERO
                    ) != 0 &&
                        value.compareTo(
                            BigDecimal.ZERO
                        ) != 0
                }

        val deltaValue =
            comparisonValue?.let { value ->
                val absoluteDeltaValue =
                    submittedMetricValue
                        .subtract(value)
                        .multiply(HUNDRED)
                        .divide(
                            value.abs(),
                            DELTA_SCALE,
                            RoundingMode.HALF_UP,
                        )
                        .abs()

                applyDeltaSign(
                    deltaValue = absoluteDeltaValue,
                    direction = metric.direction,
                    previousValue =
                        submittedMetricValue,
                    beforePreviousValue = value,
                )
            }

        return InitiativeMetricPreAnalyticsItemResponse(
            metricId = metric.id,
            name = metric.name,
            unit = metric.unit,
            value = submittedMetricValue,
            targetValue = previousValue.targetValue,
            deltaValue = deltaValue,
        )
    }

    private fun emptyMetricResponse(
        metric: MetricsDirectoryEntity,
    ): InitiativeMetricPreAnalyticsItemResponse {
        return InitiativeMetricPreAnalyticsItemResponse(
            metricId = metric.id,
            name = metric.name,
            unit = metric.unit,
            value = null,
            targetValue = null,
            deltaValue = null,
        )
    }

    private fun applyDeltaSign(
        deltaValue: BigDecimal,
        direction: String?,
        previousValue: BigDecimal,
        beforePreviousValue: BigDecimal,
    ): BigDecimal {
        val comparison =
            previousValue.compareTo(
                beforePreviousValue
            )

        val improved = when (direction) {
            MORE_IS_BETTER -> comparison >= 0
            LESS_IS_BETTER -> comparison < 0

            else -> throw IllegalStateException(
                "Unsupported metric direction: " +
                    direction
            )
        }

        return if (improved) {
            deltaValue
        } else {
            deltaValue.negate()
        }
    }

    data class MetricValueKey(
        val agentType: String,
        val metricId: UUID,
        val periodMonth: YearMonth,
    )

    private companion object {

        const val MORE_IS_BETTER =
            "more_is_better"

        const val LESS_IS_BETTER =
            "less_is_better"

        const val DELTA_SCALE = 2

        val HUNDRED: BigDecimal =
            BigDecimal.valueOf(100)
    }
}
```
