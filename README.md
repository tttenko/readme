```java
- metric-id: ${PRE_ANALYTICS_ACCURACY_METRIC_ID}
        code: ${PRE_ANALYTICS_ACCURACY_METRIC_CODE}

      - metric-id: ${PRE_ANALYTICS_CSI_METRIC_ID}
        code: ${PRE_ANALYTICS_CSI_METRIC_CODE}

      - metric-id: ${PRE_ANALYTICS_COVERAGE_METRIC_ID}
        code: ${PRE_ANALYTICS_COVERAGE_METRIC_CODE}

      - metric-id: ${PRE_ANALYTICS_SPEED_METRIC_ID}
        code: ${PRE_ANALYTICS_SPEED_METRIC_CODE}

PRE_ANALYTICS_ACCURACY_METRIC_ID: "1b28305a-3aed-4010-8e74-de752fbe1a32"
PRE_ANALYTICS_ACCURACY_METRIC_CODE: "Точность"

PRE_ANALYTICS_CSI_METRIC_ID: "2c39416b-4bfe-4121-9f85-ef863fca2b43"
PRE_ANALYTICS_CSI_METRIC_CODE: "Δ удовлетворённости"

PRE_ANALYTICS_COVERAGE_METRIC_ID: "3d4a527c-5caf-4232-af96-fa9740db3c54"
PRE_ANALYTICS_COVERAGE_METRIC_CODE: "Охват пользователей"

PRE_ANALYTICS_SPEED_METRIC_ID: "4e5b638d-6db0-4343-bfa7-ab0851ec4d65"
PRE_ANALYTICS_SPEED_METRIC_CODE: "Скорость"

@Component
class InitiativeMetricPreAnalyticsResponseBuilder {

    fun buildMetricsForAgentType(
        agentType: String,
        metricTypes: List<InitiativeMetricTypeEntity>,
        metrics: List<MetricsDirectoryEntity>,
        metricConfigurationsById:
            Map<UUID, PreAnalyticsMetricProperties>,
        valuesByKey:
            Map<MetricValueKey, InitiativeMetricValueEntity>,
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
            val metricConfiguration =
                metricConfigurationsById.getValue(
                    metric.id,
                )

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
                        periodMonth = beforePreviousMonth,
                    )
                ]

            buildMetricResponse(
                metric = metric,
                metricCode = metricConfiguration.code,
                previousValue = previousValue,
                beforePreviousValue = beforePreviousValue,
            )
        }
    }

    private fun buildMetricResponse(
        metric: MetricsDirectoryEntity,
        metricCode: String,
        previousValue: InitiativeMetricValueEntity?,
        beforePreviousValue: InitiativeMetricValueEntity?,
    ): InitiativeMetricPreAnalyticsItemResponse {
        val submittedMetricValue =
            previousValue?.metricValue

        if (
            previousValue == null ||
            submittedMetricValue == null
        ) {
            return emptyMetricResponse(
                metric = metric,
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
            metricId = metric.id,
            code = metricCode,
            name = metric.name,
            unit = metric.unit,
            value = submittedMetricValue,
            targetValue = previousValue.targetValue,
            deltaValue = deltaValue,
        )
    }

    private fun emptyMetricResponse(
        metric: MetricsDirectoryEntity,
        metricCode: String,
    ): InitiativeMetricPreAnalyticsItemResponse {
        return InitiativeMetricPreAnalyticsItemResponse(
            metricId = metric.id,
            code = metricCode,
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

        const val DELTA_SCALE = 0

        val HUNDRED: BigDecimal =
            BigDecimal.valueOf(100)
    }
}

@Service
class InitiativeMetricPreAnalyticsReader(
    private val messageProvider: MessageProvider,
    private val preAnalyticsProperties:
        PreAnalyticsProperties,
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
                message =
                    MessageFormat.format(
                        messageProvider[
                            INITIATIVE_METRIC_TYPES_NOT_FOUND
                        ],
                        initiativeId,
                    ),
            )
        }

        val configuredMetrics =
            preAnalyticsProperties.metrics

        check(
            configuredMetrics.size ==
                EXPECTED_METRICS_COUNT
        ) {
            "Exactly $EXPECTED_METRICS_COUNT " +
                "pre-analytics metrics must be configured"
        }

        check(
            configuredMetrics.all { configuration ->
                configuration.metricId != null &&
                    configuration.code.isNotBlank()
            }
        ) {
            "Pre-analytics metricId and code " +
                "must be configured"
        }

        val configuredMetricIds =
            configuredMetrics.map { configuration ->
                requireNotNull(configuration.metricId)
            }

        check(
            configuredMetricIds.toSet().size ==
                EXPECTED_METRICS_COUNT
        ) {
            "Pre-analytics metricIds must be unique"
        }

        val configuredMetricCodes =
            configuredMetrics.map { configuration ->
                configuration.code
            }

        check(
            configuredMetricCodes.toSet().size ==
                EXPECTED_METRICS_COUNT
        ) {
            "Pre-analytics metric codes must be unique"
        }

        val metricConfigurationsById =
            configuredMetrics.associateBy { configuration ->
                requireNotNull(configuration.metricId)
            }

        val metricsById =
            metricsDirectoryRepository
                .findAllById(configuredMetricIds)
                .associateBy { metric ->
                    metric.id
                }

        val missingMetricIds =
            configuredMetricIds
                .filterNot(
                    predicate = metricsById::containsKey,
                )

        check(missingMetricIds.isEmpty()) {
            "Invalid pre-analytics metrics configuration: " +
                "metrics not found=$missingMetricIds"
        }

        /*
         * Сохраняем порядок метрик,
         * заданный в конфигурации.
         */
        val metrics =
            configuredMetricIds.map { metricId ->
                metricsById.getValue(metricId)
            }

        val currentMonth =
            YearMonth.from(
                now.atZone(ZoneOffset.UTC),
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
                        configuredMetricIds.toSet(),
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
                metricConfigurationsById =
                    metricConfigurationsById,
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
                metricConfigurationsById =
                    metricConfigurationsById,
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
                            PERIOD_DISPLAY_DATE_FORMATTER,
                        ),
            metricsAutonomous = metricsAutonomous,
            metricsCopilot = metricsCopilot,
        )
    }

    private companion object {

        const val EXPECTED_METRICS_COUNT = 4

        const val METRIC_VALUES_NOT_SUBMITTED =
            "metric.not-submitted"

        const val PERIOD_DISPLAY_TEXT_PREFIX =
            "Значения на "

        val PERIOD_DISPLAY_DATE_FORMATTER:
            DateTimeFormatter =
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
```
