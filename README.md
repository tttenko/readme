```java

@Component
class InitiativeMetricPreAnalyticsValueCalculator {

    /**
     * Логика расчёта deltaValue полностью сохранена.
     */
    fun calculateDeltaValue(
        direction: String?,
        submittedMetricValue: BigDecimal,
        beforePreviousMetricValue: BigDecimal?,
    ): BigDecimal? {
        val comparisonValue =
            beforePreviousMetricValue
                ?.takeIf { value ->
                    submittedMetricValue.compareTo(BigDecimal.ZERO) != 0 &&
                        value.compareTo(BigDecimal.ZERO) != 0
                }

        return comparisonValue?.let { value ->
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
                direction = direction,
                previousValue = submittedMetricValue,
                beforePreviousValue = value,
            )
        }
    }

    fun calculateDiagramValue(
        direction: String?,
        actualValue: BigDecimal?,
        targetValue: BigDecimal?,
    ): BigDecimal? {
        if (
            actualValue == null ||
            targetValue == null ||
            targetValue.compareTo(BigDecimal.ZERO) == 0
        ) {
            return null
        }

        return when (direction) {
            MORE_IS_BETTER ->
                calculateMoreIsBetterDiagramValue(
                    actualValue = actualValue,
                    targetValue = targetValue,
                )

            LESS_IS_BETTER ->
                calculateLessIsBetterDiagramValue(
                    actualValue = actualValue,
                    targetValue = targetValue,
                )

            else ->
                throw AiInternalServerException(
                    errorCode = UNSUPPORTED_METRIC_DIRECTION,
                    message = "Unsupported metric direction: $direction",
                )
        }
    }

    private fun calculateMoreIsBetterDiagramValue(
        actualValue: BigDecimal,
        targetValue: BigDecimal,
    ): BigDecimal {
        /*
         * Проверка на 100%:
         * ФАКТ > ПЛАН.
         */
        if (actualValue > targetValue) {
            return HUNDRED
        }

        /*
         * Проверка на 0%:
         * (ФАКТ отрицательный и ПЛАН положительный)
         * или ФАКТ равен нулю.
         */
        val shouldReturnZero =
            (
                actualValue < BigDecimal.ZERO &&
                    targetValue > BigDecimal.ZERO
                ) ||
                actualValue.compareTo(BigDecimal.ZERO) == 0

        if (shouldReturnZero) {
            return BigDecimal.ZERO
        }

        /*
         * ФАКТ / ПЛАН * 100%.
         */
        return calculatePercentage(
            numerator = actualValue,
            denominator = targetValue,
        )
    }

    private fun calculateLessIsBetterDiagramValue(
        actualValue: BigDecimal,
        targetValue: BigDecimal,
    ): BigDecimal {
        /*
         * Проверка на 100%:
         * ФАКТ < ПЛАН.
         */
        if (actualValue < targetValue) {
            return HUNDRED
        }

        /*
         * Проверка на 0%:
         * (ФАКТ положительный и ПЛАН отрицательный)
         * или ФАКТ равен нулю.
         */
        val shouldReturnZero =
            (
                actualValue > BigDecimal.ZERO &&
                    targetValue < BigDecimal.ZERO
                ) ||
                actualValue.compareTo(BigDecimal.ZERO) == 0

        if (shouldReturnZero) {
            return BigDecimal.ZERO
        }

        /*
         * ПЛАН / ФАКТ * 100%.
         */
        return calculatePercentage(
            numerator = targetValue,
            denominator = actualValue,
        )
    }

    private fun calculatePercentage(
        numerator: BigDecimal,
        denominator: BigDecimal,
    ): BigDecimal {
        return numerator
            .multiply(HUNDRED)
            .divide(
                denominator,
                DIAGRAM_VALUE_SCALE,
                RoundingMode.HALF_UP,
            )
    }

    private fun applyDeltaSign(
        deltaValue: BigDecimal,
        direction: String?,
        previousValue: BigDecimal,
        beforePreviousValue: BigDecimal,
    ): BigDecimal {
        val comparison =
            previousValue.compareTo(beforePreviousValue)

        val improved =
            when (direction) {
                MORE_IS_BETTER -> comparison >= 0
                LESS_IS_BETTER -> comparison < 0

                else ->
                    throw AiInternalServerException(
                        errorCode = UNSUPPORTED_METRIC_DIRECTION,
                        message = "Unsupported metric direction: $direction",
                    )
            }

        return if (improved) {
            deltaValue
        } else {
            deltaValue.negate()
        }
    }

    private companion object {
        const val MORE_IS_BETTER = "more_is_better"
        const val LESS_IS_BETTER = "less_is_better"

        const val UNSUPPORTED_METRIC_DIRECTION =
            "unsupported.metric.direction"

        const val DELTA_SCALE = 0

        /*
         * Документация не указывает точность diagramValue.
         * Здесь возвращается значение с двумя знаками.
         */
        const val DIAGRAM_VALUE_SCALE = 2

        val HUNDRED: BigDecimal =
            BigDecimal.valueOf(100)
    }
}

@Component
class InitiativeMetricPreAnalyticsResponseBuilder(
    private val valueCalculator:
        InitiativeMetricPreAnalyticsValueCalculator,
) {

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

        val deltaValue =
            valueCalculator.calculateDeltaValue(
                direction = metric.direction,
                submittedMetricValue = submittedMetricValue,
                beforePreviousMetricValue =
                    beforePreviousValue?.metricValue,
            )

        val diagramValue =
            valueCalculator.calculateDiagramValue(
                direction = metric.direction,
                actualValue = submittedMetricValue,
                targetValue = previousValue.targetValue,
            )

        return InitiativeMetricPreAnalyticsItemResponse(
            metricId = metricId,
            code = metricCode,
            name = metric.name,
            unit = metric.unit,
            value = submittedMetricValue,
            targetValue = previousValue.targetValue,
            deltaValue = deltaValue,
            diagramValue = diagramValue,
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
            diagramValue = null,
            agentType = agentType,
        )
    }

    data class MetricValueKey(
        val agentType: String,
        val metricId: UUID,
        val periodMonth: YearMonth,
    )

    private companion object {
        const val PRE_ANALYTICS_CODE_NOT_CONFIGURED =
            "pre-analytics.metric.code-not-configured"
    }
}



```
