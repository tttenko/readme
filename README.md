```java


   @ExtendWith(MockKExtension::class)
class InitiativeMetricPreAnalyticsReaderTest {

    @MockK
    private lateinit var messageProvider: MessageProvider

    @MockK
    private lateinit var initiativeMetricTypeRepository:
        InitiativeMetricTypeRepository

    @MockK
    private lateinit var initiativeMetricValueRepository:
        InitiativeMetricValueRepository

    @MockK
    private lateinit var metricsDirectoryRepository:
        MetricsDirectoryRepository

    @MockK
    private lateinit var responseBuilder:
        InitiativeMetricPreAnalyticsResponseBuilder

    private lateinit var reader:
        InitiativeMetricPreAnalyticsReader

    private lateinit var metricTypes:
        List<InitiativeMetricTypeEntity>

    private lateinit var metrics:
        List<MetricsDirectoryEntity>

    @BeforeEach
    fun setUp() {
        reader =
            InitiativeMetricPreAnalyticsReader(
                messageProvider = messageProvider,
                initiativeMetricTypeRepository =
                    initiativeMetricTypeRepository,
                initiativeMetricValueRepository =
                    initiativeMetricValueRepository,
                metricsDirectoryRepository =
                    metricsDirectoryRepository,
                responseBuilder = responseBuilder,
            )

        metricTypes =
            listOf(
                metricType(
                    InitiativeMetricAgentType
                        .AUTONOMOUS
                        .value,
                ),
                metricType(
                    InitiativeMetricAgentType
                        .COPILOT
                        .value,
                ),
            )

        metrics = directoryMetrics()

        every {
            messageProvider[
                INITIATIVE_METRIC_TYPES_NOT_FOUND
            ]
        } returns
            "Metric types not found for initiative {0}"

        every {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdAndAgentTypeIn(
                    initiativeId = INITIATIVE_ID,
                    agentTypes = SUPPORTED_AGENT_TYPES,
                )
        } returns metricTypes

        every {
            metricsDirectoryRepository
                .findAllByPreAnalyticsTrue()
        } returns metrics

        every {
            initiativeMetricValueRepository
                .findValuesForInitiativeMetrics(
                    initiativeId = INITIATIVE_ID,
                    agentTypes = SUPPORTED_AGENT_TYPES,
                    metricDirectoryIds = any(),
                    periodFrom =
                        BEFORE_PREVIOUS_MONTH.atDay(1),
                )
        } returns emptyList()

        every {
            responseBuilder.buildMetricsForAgentType(
                agentType = any(),
                metricTypes = any(),
                metrics = any(),
                valuesByKey = any(),
                previousMonth = any(),
                beforePreviousMonth = any(),
            )
        } returns emptyList()
    }

    @Test
    fun `should throw conflict when initiative has no supported agent types`() {
        every {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdAndAgentTypeIn(
                    initiativeId = INITIATIVE_ID,
                    agentTypes = SUPPORTED_AGENT_TYPES,
                )
        } returns emptyList()

        assertThatThrownBy {
            reader.getPreAnalytics(
                initiativeId = INITIATIVE_ID,
                now = NOW,
            )
        }
            .isInstanceOf(AiConflictException::class.java)
            .hasMessage(
                "Metric types not found for initiative " +
                    INITIATIVE_ID,
            )

        verify(exactly = 0) {
            metricsDirectoryRepository
                .findAllByPreAnalyticsTrue()

            initiativeMetricValueRepository
                .findValuesForInitiativeMetrics(
                    any(),
                    any(),
                    any(),
                    any(),
                )

            responseBuilder.buildMetricsForAgentType(
                agentType = any(),
                metricTypes = any(),
                metrics = any(),
                valuesByKey = any(),
                previousMonth = any(),
                beforePreviousMonth = any(),
            )
        }
    }

    @Test
    fun `should return response without error when metric value is submitted`() {
        val metricIds =
            metrics
                .map { metric ->
                    requireNotNull(metric.id)
                }
                .toSet()

        val autonomousItem =
            responseItem(
                metric = metrics.first(),
                value = BigDecimal("120"),
            )

        val copilotItem =
            responseItem(
                metric = metrics[1],
                value = null,
            )

        every {
            responseBuilder.buildMetricsForAgentType(
                agentType =
                    InitiativeMetricAgentType
                        .AUTONOMOUS
                        .value,
                metricTypes = metricTypes,
                metrics = metrics,
                valuesByKey = emptyMap(),
                previousMonth = PREVIOUS_MONTH,
                beforePreviousMonth =
                    BEFORE_PREVIOUS_MONTH,
            )
        } returns listOf(autonomousItem)

        every {
            responseBuilder.buildMetricsForAgentType(
                agentType =
                    InitiativeMetricAgentType
                        .COPILOT
                        .value,
                metricTypes = metricTypes,
                metrics = metrics,
                valuesByKey = emptyMap(),
                previousMonth = PREVIOUS_MONTH,
                beforePreviousMonth =
                    BEFORE_PREVIOUS_MONTH,
            )
        } returns listOf(copilotItem)

        val response =
            reader.getPreAnalytics(
                initiativeId = INITIATIVE_ID,
                now = NOW,
            )

        assertThat(response.initiativeId)
            .isEqualTo(INITIATIVE_ID)

        assertThat(response.errorCode)
            .isNull()

        assertThat(response.periodDisplayText)
            .isEqualTo("Значения на 01.07.2026")

        assertThat(response.metricsAutonomous)
            .containsExactly(autonomousItem)

        assertThat(response.metricsCopilot)
            .containsExactly(copilotItem)

        verify(exactly = 1) {
            initiativeMetricValueRepository
                .findValuesForInitiativeMetrics(
                    initiativeId = INITIATIVE_ID,
                    agentTypes = SUPPORTED_AGENT_TYPES,
                    metricDirectoryIds = metricIds,
                    periodFrom =
                        LocalDate.of(2026, 5, 1),
                )
        }
    }

    @Test
    fun `should return error code when all metric values are absent`() {
        val emptyItems =
            metrics.map { metric ->
                responseItem(
                    metric = metric,
                    value = null,
                )
            }

        every {
            responseBuilder.buildMetricsForAgentType(
                agentType =
                    InitiativeMetricAgentType
                        .AUTONOMOUS
                        .value,
                metricTypes = metricTypes,
                metrics = metrics,
                valuesByKey = emptyMap(),
                previousMonth = PREVIOUS_MONTH,
                beforePreviousMonth =
                    BEFORE_PREVIOUS_MONTH,
            )
        } returns emptyItems

        every {
            responseBuilder.buildMetricsForAgentType(
                agentType =
                    InitiativeMetricAgentType
                        .COPILOT
                        .value,
                metricTypes = metricTypes,
                metrics = metrics,
                valuesByKey = emptyMap(),
                previousMonth = PREVIOUS_MONTH,
                beforePreviousMonth =
                    BEFORE_PREVIOUS_MONTH,
            )
        } returns emptyList()

        val response =
            reader.getPreAnalytics(
                initiativeId = INITIATIVE_ID,
                now = NOW,
            )

        assertThat(response.errorCode)
            .isEqualTo("metric.not-submitted")

        assertThat(response.metricsAutonomous)
            .containsExactlyElementsOf(emptyItems)

        assertThat(response.metricsCopilot)
            .isEmpty()
    }

    @Test
    fun `should not query values when pre analytics metrics are absent`() {
        every {
            metricsDirectoryRepository
                .findAllByPreAnalyticsTrue()
        } returns emptyList()

        val response =
            reader.getPreAnalytics(
                initiativeId = INITIATIVE_ID,
                now = NOW,
            )

        assertThat(response.errorCode)
            .isEqualTo("metric.not-submitted")

        assertThat(response.metricsAutonomous)
            .isEmpty()

        assertThat(response.metricsCopilot)
            .isEmpty()

        verify(exactly = 0) {
            initiativeMetricValueRepository
                .findValuesForInitiativeMetrics(
                    any(),
                    any(),
                    any(),
                    any(),
                )
        }

        verify(exactly = 2) {
            responseBuilder.buildMetricsForAgentType(
                agentType = any(),
                metricTypes = metricTypes,
                metrics = emptyList(),
                valuesByKey = emptyMap(),
                previousMonth = PREVIOUS_MONTH,
                beforePreviousMonth =
                    BEFORE_PREVIOUS_MONTH,
            )
        }
    }

    @Test
    fun `should pass only previous two months values to response builder`() {
        val metric = metrics.first()

        val previousValue =
            metricValue(
                agentType =
                    InitiativeMetricAgentType
                        .AUTONOMOUS
                        .value,
                metric = metric,
                periodMonth = PREVIOUS_MONTH,
                metricValue = BigDecimal("120"),
            )

        val beforePreviousValue =
            metricValue(
                agentType =
                    InitiativeMetricAgentType
                        .AUTONOMOUS
                        .value,
                metric = metric,
                periodMonth = BEFORE_PREVIOUS_MONTH,
                metricValue = BigDecimal("100"),
            )

        val currentValue =
            metricValue(
                agentType =
                    InitiativeMetricAgentType
                        .AUTONOMOUS
                        .value,
                metric = metric,
                periodMonth = CURRENT_MONTH,
                metricValue = BigDecimal("130"),
            )

        every {
            initiativeMetricValueRepository
                .findValuesForInitiativeMetrics(
                    initiativeId = INITIATIVE_ID,
                    agentTypes = SUPPORTED_AGENT_TYPES,
                    metricDirectoryIds = any(),
                    periodFrom =
                        BEFORE_PREVIOUS_MONTH.atDay(1),
                )
        } returns
            listOf(
                previousValue,
                beforePreviousValue,
                currentValue,
            )

        val expectedValues =
            mapOf(
                key(
                    agentType =
                        InitiativeMetricAgentType
                            .AUTONOMOUS
                            .value,
                    metricId =
                        requireNotNull(metric.id),
                    periodMonth = PREVIOUS_MONTH,
                ) to previousValue,
                key(
                    agentType =
                        InitiativeMetricAgentType
                            .AUTONOMOUS
                            .value,
                    metricId =
                        requireNotNull(metric.id),
                    periodMonth =
                        BEFORE_PREVIOUS_MONTH,
                ) to beforePreviousValue,
            )

        reader.getPreAnalytics(
            initiativeId = INITIATIVE_ID,
            now = NOW,
        )

        verify(exactly = 2) {
            responseBuilder.buildMetricsForAgentType(
                agentType = any(),
                metricTypes = metricTypes,
                metrics = metrics,
                valuesByKey = expectedValues,
                previousMonth = PREVIOUS_MONTH,
                beforePreviousMonth =
                    BEFORE_PREVIOUS_MONTH,
            )
        }
    }

    private fun directoryMetrics():
        List<MetricsDirectoryEntity> {
        return listOf(
            metric(
                id = ACCURACY_METRIC_ID,
                code = ACCURACY_CODE,
                name = "Полное название точности",
            ),
            metric(
                id = CSI_METRIC_ID,
                code = CSI_CODE,
                name = "Полное название CSI",
            ),
            metric(
                id = COVERAGE_METRIC_ID,
                code = COVERAGE_CODE,
                name = "Полное название охвата",
            ),
            metric(
                id = SPEED_METRIC_ID,
                code = SPEED_CODE,
                name = "Полное название скорости",
            ),
        )
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

    private fun metric(
        id: UUID,
        code: String,
        name: String,
    ): MetricsDirectoryEntity {
        return mockk {
            every {
                this@mockk.id
            } returns id

            every {
                this@mockk.code
            } returns code

            every {
                this@mockk.name
            } returns name

            every {
                unit
            } returns "percent"
        }
    }

    private fun metricValue(
        agentType: String,
        metric: MetricsDirectoryEntity,
        periodMonth: YearMonth,
        metricValue: BigDecimal?,
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
        }
    }

    private fun responseItem(
        metric: MetricsDirectoryEntity,
        value: BigDecimal?,
    ): InitiativeMetricPreAnalyticsItemResponse {
        return InitiativeMetricPreAnalyticsItemResponse(
            metricId = requireNotNull(metric.id),
            code = requireNotNull(metric.code),
            name = metric.name,
            unit = metric.unit,
            value = value,
            targetValue =
                value?.let {
                    BigDecimal("150")
                },
            deltaValue =
                value?.let {
                    BigDecimal("20")
                },
        )
    }

    private fun key(
        agentType: String,
        metricId: UUID,
        periodMonth: YearMonth,
    ): InitiativeMetricPreAnalyticsResponseBuilder
        .MetricValueKey {
        return InitiativeMetricPreAnalyticsResponseBuilder
            .MetricValueKey(
                agentType = agentType,
                metricId = metricId,
                periodMonth = periodMonth,
            )
    }

    private companion object {

        const val INITIATIVE_ID = 100L

        const val ACCURACY_CODE = "Точность"

        const val CSI_CODE = "Δ удовлетворённости"

        const val COVERAGE_CODE =
            "Охват пользователей"

        const val SPEED_CODE = "Скорость"

        val ACCURACY_METRIC_ID: UUID =
            UUID.fromString(
                "11111111-1111-1111-1111-111111111111",
            )

        val CSI_METRIC_ID: UUID =
            UUID.fromString(
                "22222222-2222-2222-2222-222222222222",
            )

        val COVERAGE_METRIC_ID: UUID =
            UUID.fromString(
                "33333333-3333-3333-3333-333333333333",
            )

        val SPEED_METRIC_ID: UUID =
            UUID.fromString(
                "44444444-4444-4444-4444-444444444444",
            )

        val NOW: Instant =
            Instant.parse(
                "2026-07-17T10:00:00Z",
            )

        val CURRENT_MONTH: YearMonth =
            YearMonth.of(2026, 7)

        val PREVIOUS_MONTH: YearMonth =
            YearMonth.of(2026, 6)

        val BEFORE_PREVIOUS_MONTH: YearMonth =
            YearMonth.of(2026, 5)

        val SUPPORTED_AGENT_TYPES =
            setOf(
                InitiativeMetricAgentType
                    .AUTONOMOUS
                    .value,
                InitiativeMetricAgentType
                    .COPILOT
                    .value,
            )
    }
}

@ExtendWith(MockKExtension::class)
class InitiativeMetricPreAnalyticsResponseBuilderTest {

    private lateinit var responseBuilder:
        InitiativeMetricPreAnalyticsResponseBuilder

    private lateinit var metric:
        MetricsDirectoryEntity

    private lateinit var autonomousMetricType:
        InitiativeMetricTypeEntity

    private lateinit var copilotMetricType:
        InitiativeMetricTypeEntity

    @BeforeEach
    fun setUp() {
        responseBuilder =
            InitiativeMetricPreAnalyticsResponseBuilder()

        metric =
            mockk {
                every {
                    id
                } returns METRIC_ID

                every {
                    code
                } returns METRIC_CODE

                every {
                    name
                } returns METRIC_NAME

                every {
                    unit
                } returns METRIC_UNIT

                every {
                    direction
                } returns MORE_IS_BETTER
            }

        autonomousMetricType =
            metricType(
                InitiativeMetricAgentType
                    .AUTONOMOUS
                    .value,
            )

        copilotMetricType =
            metricType(
                InitiativeMetricAgentType
                    .COPILOT
                    .value,
            )
    }

    @Test
    fun `should return empty list when requested agent type does not exist`() {
        val result =
            responseBuilder.buildMetricsForAgentType(
                agentType =
                    InitiativeMetricAgentType
                        .AUTONOMOUS
                        .value,
                metricTypes =
                    listOf(copilotMetricType),
                metrics = listOf(metric),
                valuesByKey = emptyMap(),
                previousMonth = PREVIOUS_MONTH,
                beforePreviousMonth =
                    BEFORE_PREVIOUS_MONTH,
            )

        assertThat(result).isEmpty()
    }

    @Test
    fun `should build empty response when previous value is absent`() {
        val response =
            buildResponse(
                valuesByKey = emptyMap(),
            )

        assertThat(response)
            .usingRecursiveComparison()
            .isEqualTo(
                emptyMetricResponse(),
            )
    }

    @Test
    fun `should build empty response when previous metric value is null`() {
        val previousValue =
            metricValue(
                metricValue = null,
                targetValue = BigDecimal("150"),
            )

        val response =
            buildResponse(
                valuesByKey =
                    mapOf(
                        key(
                            InitiativeMetricAgentType
                                .AUTONOMOUS
                                .value,
                            PREVIOUS_MONTH,
                        ) to previousValue,
                    ),
            )

        assertThat(response)
            .usingRecursiveComparison()
            .isEqualTo(
                emptyMetricResponse(),
            )
    }

    @Test
    fun `should return positive delta when more is better and value increased`() {
        val response =
            buildMetricResponse(
                direction = MORE_IS_BETTER,
                previousValue = BigDecimal("120"),
                beforePreviousValue =
                    BigDecimal("100"),
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
                beforePreviousValue =
                    BigDecimal("100"),
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
                beforePreviousValue =
                    BigDecimal("100"),
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
                beforePreviousValue =
                    BigDecimal("100"),
            )

        assertThat(response.deltaValue)
            .isEqualByComparingTo("-20")
    }

    @Test
    fun `should compare negative values without using absolute values`() {
        val moreIsBetterResponse =
            buildMetricResponse(
                direction = MORE_IS_BETTER,
                previousValue = BigDecimal("-2"),
                beforePreviousValue =
                    BigDecimal("-3"),
            )

        val lessIsBetterResponse =
            buildMetricResponse(
                direction = LESS_IS_BETTER,
                previousValue = BigDecimal("-3"),
                beforePreviousValue =
                    BigDecimal("-2"),
            )

        assertThat(
            moreIsBetterResponse.deltaValue,
        ).isEqualByComparingTo("33")

        assertThat(
            lessIsBetterResponse.deltaValue,
        ).isEqualByComparingTo("50")
    }

    @Test
    fun `should return zero delta when more is better and values are equal`() {
        val response =
            buildMetricResponse(
                direction = MORE_IS_BETTER,
                previousValue = BigDecimal("100"),
                beforePreviousValue =
                    BigDecimal("100"),
            )

        assertThat(response.deltaValue)
            .isEqualByComparingTo(BigDecimal.ZERO)
    }

    @Test
    fun `should return null delta when before previous value is absent`() {
        val previousValue =
            metricValue(
                metricValue = BigDecimal("120"),
                targetValue = BigDecimal("150"),
            )

        val response =
            buildResponse(
                valuesByKey =
                    mapOf(
                        key(
                            InitiativeMetricAgentType
                                .AUTONOMOUS
                                .value,
                            PREVIOUS_MONTH,
                        ) to previousValue,
                    ),
            )

        assertThat(response.code)
            .isEqualTo(METRIC_CODE)

        assertThat(response.value)
            .isEqualByComparingTo("120")

        assertThat(response.targetValue)
            .isEqualByComparingTo("150")

        assertThat(response.deltaValue)
            .isNull()
    }

    @Test
    fun `should return null delta when previous value is zero`() {
        val response =
            buildMetricResponse(
                direction = MORE_IS_BETTER,
                previousValue = BigDecimal.ZERO,
                beforePreviousValue =
                    BigDecimal("100"),
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
                beforePreviousValue =
                    BigDecimal.ZERO,
            )

        assertThat(response.value)
            .isEqualByComparingTo("100")

        assertThat(response.deltaValue)
            .isNull()
    }

    @Test
    fun `should ignore values from another agent type`() {
        val copilotValue =
            metricValue(
                metricValue = BigDecimal("120"),
                targetValue = BigDecimal("150"),
            )

        val response =
            buildResponse(
                valuesByKey =
                    mapOf(
                        key(
                            InitiativeMetricAgentType
                                .COPILOT
                                .value,
                            PREVIOUS_MONTH,
                        ) to copilotValue,
                    ),
            )

        assertThat(response)
            .usingRecursiveComparison()
            .isEqualTo(
                emptyMetricResponse(),
            )
    }

    @Test
    fun `should throw exception when metric direction is unsupported`() {
        assertThatThrownBy {
            buildMetricResponse(
                direction =
                    "unsupported_direction",
                previousValue =
                    BigDecimal("120"),
                beforePreviousValue =
                    BigDecimal("100"),
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

    @Test
    fun `should throw exception when pre analytics metric code is absent`() {
        every {
            metric.code
        } returns null

        assertThatThrownBy {
            buildResponse(
                valuesByKey = emptyMap(),
            )
        }
            .isInstanceOf(
                AiInternalServerException::class.java,
            )
            .hasMessage(
                "Code is not configured for " +
                    "pre-analytics metric $METRIC_ID",
            )
    }

    private fun buildMetricResponse(
        direction: String,
        previousValue: BigDecimal,
        beforePreviousValue: BigDecimal,
    ): InitiativeMetricPreAnalyticsItemResponse {
        every {
            metric.direction
        } returns direction

        val previousMetricValue =
            metricValue(
                metricValue = previousValue,
                targetValue = BigDecimal("150"),
            )

        val beforePreviousMetricValue =
            metricValue(
                metricValue = beforePreviousValue,
                targetValue = BigDecimal("150"),
            )

        return buildResponse(
            valuesByKey =
                mapOf(
                    key(
                        InitiativeMetricAgentType
                            .AUTONOMOUS
                            .value,
                        PREVIOUS_MONTH,
                    ) to previousMetricValue,
                    key(
                        InitiativeMetricAgentType
                            .AUTONOMOUS
                            .value,
                        BEFORE_PREVIOUS_MONTH,
                    ) to beforePreviousMetricValue,
                ),
        )
    }

    private fun buildResponse(
        valuesByKey:
            Map<
                InitiativeMetricPreAnalyticsResponseBuilder
                    .MetricValueKey,
                InitiativeMetricValueEntity
            >,
    ): InitiativeMetricPreAnalyticsItemResponse {
        return responseBuilder
            .buildMetricsForAgentType(
                agentType =
                    InitiativeMetricAgentType
                        .AUTONOMOUS
                        .value,
                metricTypes =
                    listOf(autonomousMetricType),
                metrics = listOf(metric),
                valuesByKey = valuesByKey,
                previousMonth = PREVIOUS_MONTH,
                beforePreviousMonth =
                    BEFORE_PREVIOUS_MONTH,
            )
            .single()
    }

    private fun emptyMetricResponse():
        InitiativeMetricPreAnalyticsItemResponse {
        return InitiativeMetricPreAnalyticsItemResponse(
            metricId = METRIC_ID,
            code = METRIC_CODE,
            name = METRIC_NAME,
            unit = METRIC_UNIT,
            value = null,
            targetValue = null,
            deltaValue = null,
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
        metricValue: BigDecimal?,
        targetValue: BigDecimal?,
    ): InitiativeMetricValueEntity {
        return mockk {
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

        const val METRIC_CODE = "Точность"

        const val METRIC_NAME =
            "Полное название метрики точности"

        const val METRIC_UNIT = "percent"

        const val MORE_IS_BETTER =
            "more_is_better"

        const val LESS_IS_BETTER =
            "less_is_better"
    }
}
```
