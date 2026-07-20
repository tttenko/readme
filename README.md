```java
@ExtendWith(MockKExtension::class)
class InitiativeMetricPreAnalyticsResponseBuilderTest {

    private val responseBuilder =
        InitiativeMetricPreAnalyticsResponseBuilder()

    @Test
    fun `should return empty list when requested agent type does not exist`() {
        val metricType =
            metricType(
                agentType =
                    InitiativeMetricAgentType.COPILOT.value,
            )

        val result =
            responseBuilder.buildMetricsForAgentType(
                agentType =
                    InitiativeMetricAgentType.AUTONOMOUS.value,
                metricTypes = listOf(metricType),
                metrics = listOf(metric()),
                metricConfigurationsById =
                    metricConfigurationsById(),
                valuesByKey = emptyMap(),
                previousMonth = PREVIOUS_MONTH,
                beforePreviousMonth =
                    BEFORE_PREVIOUS_MONTH,
            )

        assertThat(result).isEmpty()
    }

    @Test
    fun `should build empty metric response when previous value is absent`() {
        val metric =
            metric(
                direction = MORE_IS_BETTER,
            )

        val result =
            responseBuilder.buildMetricsForAgentType(
                agentType =
                    InitiativeMetricAgentType.AUTONOMOUS.value,
                metricTypes =
                    listOf(
                        metricType(
                            InitiativeMetricAgentType
                                .AUTONOMOUS
                                .value,
                        ),
                    ),
                metrics = listOf(metric),
                metricConfigurationsById =
                    metricConfigurationsById(),
                valuesByKey = emptyMap(),
                previousMonth = PREVIOUS_MONTH,
                beforePreviousMonth =
                    BEFORE_PREVIOUS_MONTH,
            )

        assertThat(result).hasSize(1)

        assertThat(result.single())
            .usingRecursiveComparison()
            .isEqualTo(
                InitiativeMetricPreAnalyticsItemResponse(
                    metricId = METRIC_ID,
                    code = METRIC_CODE,
                    name = METRIC_NAME,
                    unit = METRIC_UNIT,
                    value = null,
                    targetValue = null,
                    deltaValue = null,
                ),
            )
    }

    @Test
    fun `should build empty metric response when previous metric value is null`() {
        val metric =
            metric(
                direction = MORE_IS_BETTER,
            )

        val previousValue =
            metricValue(
                agentType =
                    InitiativeMetricAgentType
                        .AUTONOMOUS
                        .value,
                metric = metric,
                periodMonth = PREVIOUS_MONTH,
                metricValue = null,
                targetValue = BigDecimal("150"),
            )

        val result =
            responseBuilder.buildMetricsForAgentType(
                agentType =
                    InitiativeMetricAgentType
                        .AUTONOMOUS
                        .value,
                metricTypes =
                    listOf(
                        metricType(
                            InitiativeMetricAgentType
                                .AUTONOMOUS
                                .value,
                        ),
                    ),
                metrics = listOf(metric),
                metricConfigurationsById =
                    metricConfigurationsById(),
                valuesByKey =
                    mapOf(
                        key(
                            agentType =
                                InitiativeMetricAgentType
                                    .AUTONOMOUS
                                    .value,
                            periodMonth = PREVIOUS_MONTH,
                        ) to previousValue,
                    ),
                previousMonth = PREVIOUS_MONTH,
                beforePreviousMonth =
                    BEFORE_PREVIOUS_MONTH,
            )

        assertThat(result.single())
            .usingRecursiveComparison()
            .isEqualTo(
                InitiativeMetricPreAnalyticsItemResponse(
                    metricId = METRIC_ID,
                    code = METRIC_CODE,
                    name = METRIC_NAME,
                    unit = METRIC_UNIT,
                    value = null,
                    targetValue = null,
                    deltaValue = null,
                ),
            )
    }

    @Test
    fun `should return positive delta when more is better and value increased`() {
        val response =
            buildMetricResponse(
                direction = MORE_IS_BETTER,
                previousValue = BigDecimal("120"),
                beforePreviousValue = BigDecimal("100"),
            )

        assertThat(response.metricId)
            .isEqualTo(METRIC_ID)

        assertThat(response.code)
            .isEqualTo(METRIC_CODE)

        assertThat(response.name)
            .isEqualTo(METRIC_NAME)

        assertThat(response.value)
            .isEqualByComparingTo("120")

        assertThat(response.targetValue)
            .isEqualByComparingTo("150")

        assertThat(response.deltaValue)
            .isEqualByComparingTo("20")
    }

    @Test
    fun `should return negative delta when more is better and value decreased`() {
        val response =
            buildMetricResponse(
                direction = MORE_IS_BETTER,
                previousValue = BigDecimal("80"),
                beforePreviousValue = BigDecimal("100"),
            )

        assertThat(response.deltaValue)
            .isEqualByComparingTo("-20")
    }

    @Test
    fun `should return positive delta when less is better and value decreased`() {
        val response =
            buildMetricResponse(
                direction = LESS_IS_BETTER,
                previousValue = BigDecimal("80"),
                beforePreviousValue = BigDecimal("100"),
            )

        assertThat(response.deltaValue)
            .isEqualByComparingTo("20")
    }

    @Test
    fun `should return negative delta when less is better and value increased`() {
        val response =
            buildMetricResponse(
                direction = LESS_IS_BETTER,
                previousValue = BigDecimal("120"),
                beforePreviousValue = BigDecimal("100"),
            )

        assertThat(response.deltaValue)
            .isEqualByComparingTo("-20")
    }

    @Test
    fun `should compare negative values without using absolute value`() {
        val moreIsBetterResponse =
            buildMetricResponse(
                direction = MORE_IS_BETTER,
                previousValue = BigDecimal("-2"),
                beforePreviousValue = BigDecimal("-3"),
            )

        val lessIsBetterResponse =
            buildMetricResponse(
                direction = LESS_IS_BETTER,
                previousValue = BigDecimal("-3"),
                beforePreviousValue = BigDecimal("-2"),
            )

        assertThat(moreIsBetterResponse.deltaValue)
            .isEqualByComparingTo("33")

        assertThat(lessIsBetterResponse.deltaValue)
            .isEqualByComparingTo("50")
    }

    @Test
    fun `should return zero delta when more is better and values are equal`() {
        val response =
            buildMetricResponse(
                direction = MORE_IS_BETTER,
                previousValue = BigDecimal("100"),
                beforePreviousValue = BigDecimal("100"),
            )

        assertThat(response.deltaValue)
            .isEqualByComparingTo(BigDecimal.ZERO)
    }

    @Test
    fun `should return null delta when before previous value is absent`() {
        val metric =
            metric(
                direction = MORE_IS_BETTER,
            )

        val previousValue =
            metricValue(
                agentType =
                    InitiativeMetricAgentType
                        .AUTONOMOUS
                        .value,
                metric = metric,
                periodMonth = PREVIOUS_MONTH,
                metricValue = BigDecimal("120"),
                targetValue = BigDecimal("150"),
            )

        val result =
            responseBuilder.buildMetricsForAgentType(
                agentType =
                    InitiativeMetricAgentType
                        .AUTONOMOUS
                        .value,
                metricTypes =
                    listOf(
                        metricType(
                            InitiativeMetricAgentType
                                .AUTONOMOUS
                                .value,
                        ),
                    ),
                metrics = listOf(metric),
                metricConfigurationsById =
                    metricConfigurationsById(),
                valuesByKey =
                    mapOf(
                        key(
                            agentType =
                                InitiativeMetricAgentType
                                    .AUTONOMOUS
                                    .value,
                            periodMonth = PREVIOUS_MONTH,
                        ) to previousValue,
                    ),
                previousMonth = PREVIOUS_MONTH,
                beforePreviousMonth =
                    BEFORE_PREVIOUS_MONTH,
            )

        assertThat(result.single().code)
            .isEqualTo(METRIC_CODE)

        assertThat(result.single().value)
            .isEqualByComparingTo("120")

        assertThat(result.single().targetValue)
            .isEqualByComparingTo("150")

        assertThat(result.single().deltaValue)
            .isNull()
    }

    @Test
    fun `should return null delta when previous value is zero`() {
        val response =
            buildMetricResponse(
                direction = MORE_IS_BETTER,
                previousValue = BigDecimal.ZERO,
                beforePreviousValue = BigDecimal("100"),
            )

        assertThat(response.value)
            .isEqualByComparingTo(BigDecimal.ZERO)

        assertThat(response.deltaValue)
            .isNull()
    }

    @Test
    fun `should return null delta when before previous value is zero`() {
        val response =
            buildMetricResponse(
                direction = MORE_IS_BETTER,
                previousValue = BigDecimal("100"),
                beforePreviousValue = BigDecimal.ZERO,
            )

        assertThat(response.value)
            .isEqualByComparingTo("100")

        assertThat(response.deltaValue)
            .isNull()
    }

    @Test
    fun `should ignore values from another agent type`() {
        val metric =
            metric(
                direction = MORE_IS_BETTER,
            )

        val copilotValue =
            metricValue(
                agentType =
                    InitiativeMetricAgentType
                        .COPILOT
                        .value,
                metric = metric,
                periodMonth = PREVIOUS_MONTH,
                metricValue = BigDecimal("120"),
                targetValue = BigDecimal("150"),
            )

        val result =
            responseBuilder.buildMetricsForAgentType(
                agentType =
                    InitiativeMetricAgentType
                        .AUTONOMOUS
                        .value,
                metricTypes =
                    listOf(
                        metricType(
                            InitiativeMetricAgentType
                                .AUTONOMOUS
                                .value,
                        ),
                    ),
                metrics = listOf(metric),
                metricConfigurationsById =
                    metricConfigurationsById(),
                valuesByKey =
                    mapOf(
                        key(
                            agentType =
                                InitiativeMetricAgentType
                                    .COPILOT
                                    .value,
                            periodMonth = PREVIOUS_MONTH,
                        ) to copilotValue,
                    ),
                previousMonth = PREVIOUS_MONTH,
                beforePreviousMonth =
                    BEFORE_PREVIOUS_MONTH,
            )

        assertThat(result.single().metricId)
            .isEqualTo(METRIC_ID)

        assertThat(result.single().code)
            .isEqualTo(METRIC_CODE)

        assertThat(result.single().name)
            .isEqualTo(METRIC_NAME)

        assertThat(result.single().value)
            .isNull()

        assertThat(result.single().targetValue)
            .isNull()

        assertThat(result.single().deltaValue)
            .isNull()
    }

    @Test
    fun `should throw exception when metric direction is unsupported`() {
        assertThatThrownBy {
            buildMetricResponse(
                direction = "unsupported_direction",
                previousValue = BigDecimal("120"),
                beforePreviousValue = BigDecimal("100"),
            )
        }
            .isInstanceOf(
                AiInternalServerException::class.java,
            )
            .hasMessage(
                "Unsupported metric direction: " +
                    "unsupported_direction",
            )
    }

    private fun buildMetricResponse(
        direction: String,
        previousValue: BigDecimal,
        beforePreviousValue: BigDecimal,
    ): InitiativeMetricPreAnalyticsItemResponse {
        val metric =
            metric(
                direction = direction,
            )

        val previousMetricValue =
            metricValue(
                agentType =
                    InitiativeMetricAgentType
                        .AUTONOMOUS
                        .value,
                metric = metric,
                periodMonth = PREVIOUS_MONTH,
                metricValue = previousValue,
                targetValue = BigDecimal("150"),
            )

        val beforePreviousMetricValue =
            metricValue(
                agentType =
                    InitiativeMetricAgentType
                        .AUTONOMOUS
                        .value,
                metric = metric,
                periodMonth = BEFORE_PREVIOUS_MONTH,
                metricValue = beforePreviousValue,
                targetValue = BigDecimal("150"),
            )

        return responseBuilder
            .buildMetricsForAgentType(
                agentType =
                    InitiativeMetricAgentType
                        .AUTONOMOUS
                        .value,
                metricTypes =
                    listOf(
                        metricType(
                            InitiativeMetricAgentType
                                .AUTONOMOUS
                                .value,
                        ),
                    ),
                metrics = listOf(metric),
                metricConfigurationsById =
                    metricConfigurationsById(),
                valuesByKey =
                    mapOf(
                        key(
                            agentType =
                                InitiativeMetricAgentType
                                    .AUTONOMOUS
                                    .value,
                            periodMonth = PREVIOUS_MONTH,
                        ) to previousMetricValue,
                        key(
                            agentType =
                                InitiativeMetricAgentType
                                    .AUTONOMOUS
                                    .value,
                            periodMonth =
                                BEFORE_PREVIOUS_MONTH,
                        ) to beforePreviousMetricValue,
                    ),
                previousMonth = PREVIOUS_MONTH,
                beforePreviousMonth =
                    BEFORE_PREVIOUS_MONTH,
            )
            .single()
    }

    private fun metricConfigurationsById():
        Map<UUID, PreAnalyticsMetricProperties> {
        return mapOf(
            METRIC_ID to
                PreAnalyticsMetricProperties(
                    metricId = METRIC_ID,
                    code = METRIC_CODE,
                ),
        )
    }

    private fun key(
        agentType: String,
        periodMonth: YearMonth,
    ): InitiativeMetricPreAnalyticsResponseBuilder
        .MetricValueKey {
        return InitiativeMetricPreAnalyticsResponseBuilder
            .MetricValueKey(
                agentType = agentType,
                metricId = METRIC_ID,
                periodMonth = periodMonth,
            )
    }

    private fun metric(
        direction: String = MORE_IS_BETTER,
    ): MetricsDirectoryEntity {
        return mockk {
            every {
                id
            } returns METRIC_ID

            every {
                name
            } returns METRIC_NAME

            every {
                unit
            } returns METRIC_UNIT

            every {
                this@mockk.direction
            } returns direction
        }
    }

    private fun metricType(
        agentType: String,
    ): InitiativeMetricTypeEntity {
        return mockk {
            every {
                this@mockk.agentType
            } returns agentType
        }
    }

    private fun metricValue(
        agentType: String,
        metric: MetricsDirectoryEntity,
        periodMonth: YearMonth,
        metricValue: BigDecimal?,
        targetValue: BigDecimal?,
    ): InitiativeMetricValueEntity {
        val initiativeMetricType =
            metricType(agentType)

        return mockk {
            every {
                this@mockk.initiativeMetricType
            } returns initiativeMetricType

            every {
                metricDirectory
            } returns metric

            every {
                this@mockk.periodMonth
            } returns periodMonth.atDay(1)

            every {
                this@mockk.metricValue
            } returns metricValue

            every {
                this@mockk.targetValue
            } returns targetValue
        }
    }

    private companion object {

        val METRIC_ID: UUID =
            UUID.fromString(
                "a860b390-b739-48e6-a694-e96582eb4e95",
            )

        val PREVIOUS_MONTH: YearMonth =
            YearMonth.of(2026, 6)

        val BEFORE_PREVIOUS_MONTH: YearMonth =
            YearMonth.of(2026, 5)

        const val METRIC_CODE =
            "Точность"

        const val METRIC_NAME =
            "Полное название метрики точности"

        const val METRIC_UNIT =
            "percent"

        const val MORE_IS_BETTER =
            "more_is_better"

        const val LESS_IS_BETTER =
            "less_is_better"
    }
}
```
