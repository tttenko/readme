```java


@Component
class InitiativeMetricPreAnalyticsResponseBuilder {

    fun buildMetricsForAgentType(
        agentType: String,
        metricTypes: List<InitiativeMetricTypeEntity>,
        metrics: List<MetricsDirectoryEntity>,
        valuesByKey: Map<MetricValueKey, InitiativeMetricValueEntity>,
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
            val metricId =
                requireNotNull(metric.id)

            val previousValue =
                valuesByKey[
                    MetricValueKey(
                        agentType = agentType,
                        metricId = metricId,
                        periodMonth = previousMonth,
                    )
                ]

            val beforePreviousValue =
                valuesByKey[
                    MetricValueKey(
                        agentType = agentType,
                        metricId = metricId,
                        periodMonth = beforePreviousMonth,
                    )
                ]

            buildMetricResponse(
                agentType = agentType,
                metric = metric,
                metricId = metricId,
                previousValue = previousValue,
                beforePreviousValue = beforePreviousValue,
            )
        }
    }

    private fun buildMetricResponse(
        agentType: String,
        metric: MetricsDirectoryEntity,
        metricId: UUID,
        previousValue: InitiativeMetricValueEntity?,
        beforePreviousValue: InitiativeMetricValueEntity?,
    ): InitiativeMetricPreAnalyticsItemResponse {
        val metricCode =
            metric.code
                ?.takeIf(String::isNotBlank)
                ?: throw AiInternalServerException(
                    errorCode =
                        PRE_ANALYTICS_CODE_NOT_CONFIGURED,
                    message =
                        "Code is not configured for " +
                            "pre-analytics metric $metricId",
                )

        val submittedMetricValue =
            previousValue?.metricValue

        if (
            previousValue == null ||
            submittedMetricValue == null
        ) {
            return emptyMetricResponse(
                agentType = agentType,
                metric = metric,
                metricId = metricId,
                metricCode = metricCode,
            )
        }

        val comparisonValue =
            beforePreviousValue
                ?.metricValue
                ?.takeIf { value ->
                    submittedMetricValue.compareTo(
                        BigDecimal.ZERO,
                    ) != 0 &&
                        value.compareTo(
                            BigDecimal.ZERO,
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
                    previousValue = submittedMetricValue,
                    beforePreviousValue = value,
                )
            }

        return InitiativeMetricPreAnalyticsItemResponse(
            metricId = metricId,
            code = metricCode,
            name = metric.name,
            unit = metric.unit,
            value = submittedMetricValue,
            targetValue = previousValue.targetValue,
            deltaValue = deltaValue,
            agentType = agentType,
        )
    }

    private fun emptyMetricResponse(
        agentType: String,
        metric: MetricsDirectoryEntity,
        metricId: UUID,
        metricCode: String,
    ): InitiativeMetricPreAnalyticsItemResponse {
        return InitiativeMetricPreAnalyticsItemResponse(
            metricId = metricId,
            code = metricCode,
            name = metric.name,
            unit = metric.unit,
            value = null,
            targetValue = null,
            deltaValue = null,
            agentType = agentType,
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
                beforePreviousValue,
            )

        val improved =
            when (direction) {
                MORE_IS_BETTER ->
                    comparison >= 0

                LESS_IS_BETTER ->
                    comparison < 0

                else ->
                    throw AiInternalServerException(
                        errorCode =
                            UNSUPPORTED_METRIC_DIRECTION,
                        message =
                            "Unsupported metric direction: " +
                                direction,
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

        const val UNSUPPORTED_METRIC_DIRECTION =
            "unsupported.metric.direction"

        const val PRE_ANALYTICS_CODE_NOT_CONFIGURED =
            "pre-analytics.metric.code-not-configured"

        const val DELTA_SCALE = 0

        val HUNDRED: BigDecimal =
            BigDecimal.valueOf(100)
    }
}

@Service
class InitiativeMetricPreAnalyticsReader(
    private val messageProvider: MessageProvider,
    private val initiativeMetricTypeRepository: InitiativeMetricTypeRepository,
    private val initiativeMetricValueRepository: InitiativeMetricValueRepository,
    private val metricsDirectoryRepository: MetricsDirectoryRepository,
    private val responseBuilder: InitiativeMetricPreAnalyticsResponseBuilder,
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
                errorCode = INITIATIVE_METRIC_TYPES_NOT_FOUND,
                message = MessageFormat.format(
                    messageProvider[INITIATIVE_METRIC_TYPES_NOT_FOUND],
                    initiativeId,
                ),
            )
        }

        val metrics =
            metricsDirectoryRepository.findAllByIsPreAnalyticsTrue()

        val metricIds =
            metrics
                .map { metric ->
                    requireNotNull(metric.id)
                }
                .toSet()

        val currentMonth =
            YearMonth.from(
                now.atZone(ZoneOffset.UTC),
            )

        val previousMonth =
            currentMonth.minusMonths(1)

        val beforePreviousMonth =
            currentMonth.minusMonths(2)

        val metricValues =
            if (metricIds.isEmpty()) {
                emptyList()
            } else {
                initiativeMetricValueRepository
                    .findValuesForInitiativeMetrics(
                        initiativeId = initiativeId,
                        agentTypes = SUPPORTED_AGENT_TYPES,
                        metricDirectoryIds = metricIds,
                        periodFrom = beforePreviousMonth.atDay(1),
                    )
            }

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
                            metricId =
                                requireNotNull(
                                    metricValue
                                        .metricDirectory
                                        ?.id,
                                ),
                            periodMonth =
                                YearMonth.from(
                                    requireNotNull(
                                        metricValue.periodMonth,
                                    ),
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
                beforePreviousMonth = beforePreviousMonth,
            )

        val metricsCopilot =
            responseBuilder.buildMetricsForAgentType(
                agentType = COPILOT.value,
                metricTypes = metricTypes,
                metrics = metrics,
                valuesByKey = valuesByKey,
                previousMonth = previousMonth,
                beforePreviousMonth = beforePreviousMonth,
            )

        val responseMetrics =
            metricsAutonomous + metricsCopilot

        val hasSubmittedValues =
            responseMetrics.any { metric ->
                metric.value != null
            }

        return InitiativeMetricPreAnalyticsResponse(
            initiativeId = initiativeId,
            errorCode =
                METRIC_VALUES_NOT_SUBMITTED
                    .takeUnless { hasSubmittedValues },
            periodDisplayText =
                PERIOD_DISPLAY_TEXT_PREFIX +
                    currentMonth
                        .atDay(1)
                        .format(
                            PERIOD_DISPLAY_DATE_FORMATTER,
                        ),
            metrics = responseMetrics,
        )
    }

    private companion object {
        const val METRIC_VALUES_NOT_SUBMITTED =
            "metric.not-submitted"

        const val PERIOD_DISPLAY_TEXT_PREFIX =
            "Значения на "

        val PERIOD_DISPLAY_DATE_FORMATTER: DateTimeFormatter =
            DateTimeFormatter.ofPattern(
                "dd.MM.yyyy",
            )

        val SUPPORTED_AGENT_TYPES =
            setOf(
                AUTONOMOUS.value,
                COPILOT.value,
            )
    }
}

data class InitiativeMetricPreAnalyticsResponse(
    val initiativeId: Long,
    val errorCode: String?,
    val periodDisplayText: String,
    val metrics: List<InitiativeMetricPreAnalyticsItemResponse>,
)

data class InitiativeMetricPreAnalyticsItemResponse(
    val metricId: UUID,
    val code: String?,
    val name: String?,
    val unit: String?,
    val value: BigDecimal?,
    val targetValue: BigDecimal?,
    val deltaValue: BigDecimal?,
    val agentType: String,
)
```
