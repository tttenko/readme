```java
@ExtendWith(MockKExtension::class)
class InitiativeMetricPreAnalyticsResponseBuilderTest {

    private val responseBuilder =
        InitiativeMetricPreAnalyticsResponseBuilder()

    @Test
    fun `should return empty list when requested agent type does not exist`() {
        val metricType = metricType(
            agentType = COPILOT.value,
        )

        val result =
            responseBuilder.buildMetricsForAgentType(
                agentType = AUTONOMOUS.value,
                metricTypes = listOf(metricType),
                metrics = listOf(metric()),
                valuesByKey = emptyMap(),
                previousMonth = PREVIOUS_MONTH,
                beforePreviousMonth = BEFORE_PREVIOUS_MONTH,
            )

        assertThat(result).isEmpty()
    }

    @Test
    fun `should build empty metric response when previous value is absent`() {
        val metric = metric(
            direction = MORE_IS_BETTER,
        )

        val result =
            responseBuilder.buildMetricsForAgentType(
                agentType = AUTONOMOUS.value,
                metricTypes = listOf(
                    metricType(AUTONOMOUS.value)
                ),
                metrics = listOf(metric),
                valuesByKey = emptyMap(),
                previousMonth = PREVIOUS_MONTH,
                beforePreviousMonth = BEFORE_PREVIOUS_MONTH,
            )

        assertThat(result).hasSize(1)

        assertThat(result.single())
            .usingRecursiveComparison()
            .isEqualTo(
                InitiativeMetricPreAnalyticsItemResponse(
                    metricId = METRIC_ID,
                    name = METRIC_NAME,
                    unit = METRIC_UNIT,
                    value = null,
                    targetValue = null,
                    deltaValue = null,
                )
            )
    }

    @Test
    fun `should build empty metric response when previous metric value is null`() {
        val metric = metric(
            direction = MORE_IS_BETTER,
        )

        val previousValue = metricValue(
            agentType = AUTONOMOUS.value,
            metric = metric,
            periodMonth = PREVIOUS_MONTH,
            metricValue = null,
            targetValue = BigDecimal("150"),
        )

        val result =
            responseBuilder.buildMetricsForAgentType(
                agentType = AUTONOMOUS.value,
                metricTypes = listOf(
                    metricType(AUTONOMOUS.value)
                ),
                metrics = listOf(metric),
                valuesByKey = mapOf(
                    key(
                        agentType = AUTONOMOUS.value,
                        periodMonth = PREVIOUS_MONTH,
                    ) to previousValue
                ),
                previousMonth = PREVIOUS_MONTH,
                beforePreviousMonth = BEFORE_PREVIOUS_MONTH,
            )

        assertThat(result.single())
            .usingRecursiveComparison()
            .isEqualTo(
                InitiativeMetricPreAnalyticsItemResponse(
                    metricId = METRIC_ID,
                    name = METRIC_NAME,
                    unit = METRIC_UNIT,
                    value = null,
                    targetValue = null,
                    deltaValue = null,
                )
            )
    }

    @Test
    fun `should return positive delta when more is better and value increased`() {
        val response = buildMetricResponse(
            direction = MORE_IS_BETTER,
            previousValue = BigDecimal("120"),
            beforePreviousValue = BigDecimal("100"),
        )

        assertThat(response.value)
            .isEqualByComparingTo("120")

        assertThat(response.targetValue)
            .isEqualByComparingTo("150")

        assertThat(response.deltaValue)
            .isEqualByComparingTo("20.00")
    }

    @Test
    fun `should return negative delta when more is better and value decreased`() {
        val response = buildMetricResponse(
            direction = MORE_IS_BETTER,
            previousValue = BigDecimal("80"),
            beforePreviousValue = BigDecimal("100"),
        )

        assertThat(response.deltaValue)
            .isEqualByComparingTo("-20.00")
    }

    @Test
    fun `should return positive delta when less is better and value decreased`() {
        val response = buildMetricResponse(
            direction = LESS_IS_BETTER,
            previousValue = BigDecimal("80"),
            beforePreviousValue = BigDecimal("100"),
        )

        assertThat(response.deltaValue)
            .isEqualByComparingTo("20.00")
    }

    @Test
    fun `should return negative delta when less is better and value increased`() {
        val response = buildMetricResponse(
            direction = LESS_IS_BETTER,
            previousValue = BigDecimal("120"),
            beforePreviousValue = BigDecimal("100"),
        )

        assertThat(response.deltaValue)
            .isEqualByComparingTo("-20.00")
    }

    @Test
    fun `should compare negative values without using absolute value`() {
        val moreIsBetterResponse = buildMetricResponse(
            direction = MORE_IS_BETTER,
            previousValue = BigDecimal("-2"),
            beforePreviousValue = BigDecimal("-3"),
        )

        val lessIsBetterResponse = buildMetricResponse(
            direction = LESS_IS_BETTER,
            previousValue = BigDecimal("-3"),
            beforePreviousValue = BigDecimal("-2"),
        )

        assertThat(moreIsBetterResponse.deltaValue)
            .isEqualByComparingTo("33.33")

        assertThat(lessIsBetterResponse.deltaValue)
            .isEqualByComparingTo("50.00")
    }

    @Test
    fun `should return zero delta when more is better and values are equal`() {
        val response = buildMetricResponse(
            direction = MORE_IS_BETTER,
            previousValue = BigDecimal("100"),
            beforePreviousValue = BigDecimal("100"),
        )

        assertThat(response.deltaValue)
            .isEqualByComparingTo(BigDecimal.ZERO)
    }

    @Test
    fun `should return null delta when before previous value is absent`() {
        val metric = metric(
            direction = MORE_IS_BETTER,
        )

        val previousValue = metricValue(
            agentType = AUTONOMOUS.value,
            metric = metric,
            periodMonth = PREVIOUS_MONTH,
            metricValue = BigDecimal("120"),
            targetValue = BigDecimal("150"),
        )

        val result =
            responseBuilder.buildMetricsForAgentType(
                agentType = AUTONOMOUS.value,
                metricTypes = listOf(
                    metricType(AUTONOMOUS.value)
                ),
                metrics = listOf(metric),
                valuesByKey = mapOf(
                    key(
                        agentType = AUTONOMOUS.value,
                        periodMonth = PREVIOUS_MONTH,
                    ) to previousValue
                ),
                previousMonth = PREVIOUS_MONTH,
                beforePreviousMonth = BEFORE_PREVIOUS_MONTH,
            )

        assertThat(result.single().value)
            .isEqualByComparingTo("120")

        assertThat(result.single().targetValue)
            .isEqualByComparingTo("150")

        assertThat(result.single().deltaValue).isNull()
    }

    @Test
    fun `should return null delta when one of compared values is zero`() {
        val response = buildMetricResponse(
            direction = MORE_IS_BETTER,
            previousValue = BigDecimal.ZERO,
            beforePreviousValue = BigDecimal("100"),
        )

        assertThat(response.value)
            .isEqualByComparingTo(BigDecimal.ZERO)

        assertThat(response.deltaValue).isNull()
    }

    @Test
    fun `should ignore values from another agent type`() {
        val metric = metric(
            direction = MORE_IS_BETTER,
        )

        val copilotValue = metricValue(
            agentType = COPILOT.value,
            metric = metric,
            periodMonth = PREVIOUS_MONTH,
            metricValue = BigDecimal("120"),
            targetValue = BigDecimal("150"),
        )

        val result =
            responseBuilder.buildMetricsForAgentType(
                agentType = AUTONOMOUS.value,
                metricTypes = listOf(
                    metricType(AUTONOMOUS.value)
                ),
                metrics = listOf(metric),
                valuesByKey = mapOf(
                    key(
                        agentType = COPILOT.value,
                        periodMonth = PREVIOUS_MONTH,
                    ) to copilotValue
                ),
                previousMonth = PREVIOUS_MONTH,
                beforePreviousMonth = BEFORE_PREVIOUS_MONTH,
            )

        assertThat(result.single().value).isNull()
        assertThat(result.single().targetValue).isNull()
        assertThat(result.single().deltaValue).isNull()
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
            .isInstanceOf(IllegalStateException::class.java)
            .hasMessage(
                "Unsupported metric direction: unsupported_direction"
            )
    }

    private fun buildMetricResponse(
        direction: String,
        previousValue: BigDecimal,
        beforePreviousValue: BigDecimal,
    ): InitiativeMetricPreAnalyticsItemResponse {
        val metric = metric(
            direction = direction,
        )

        val previousMetricValue = metricValue(
            agentType = AUTONOMOUS.value,
            metric = metric,
            periodMonth = PREVIOUS_MONTH,
            metricValue = previousValue,
            targetValue = BigDecimal("150"),
        )

        val beforePreviousMetricValue = metricValue(
            agentType = AUTONOMOUS.value,
            metric = metric,
            periodMonth = BEFORE_PREVIOUS_MONTH,
            metricValue = beforePreviousValue,
            targetValue = BigDecimal("150"),
        )

        return responseBuilder
            .buildMetricsForAgentType(
                agentType = AUTONOMOUS.value,
                metricTypes = listOf(
                    metricType(AUTONOMOUS.value)
                ),
                metrics = listOf(metric),
                valuesByKey = mapOf(
                    key(
                        agentType = AUTONOMOUS.value,
                        periodMonth = PREVIOUS_MONTH,
                    ) to previousMetricValue,
                    key(
                        agentType = AUTONOMOUS.value,
                        periodMonth = BEFORE_PREVIOUS_MONTH,
                    ) to beforePreviousMetricValue,
                ),
                previousMonth = PREVIOUS_MONTH,
                beforePreviousMonth = BEFORE_PREVIOUS_MONTH,
            )
            .single()
    }

    private fun key(
        agentType: String,
        periodMonth: YearMonth,
    ) = InitiativeMetricPreAnalyticsResponseBuilder
        .MetricValueKey(
            agentType = agentType,
            metricId = METRIC_ID,
            periodMonth = periodMonth,
        )

    private fun metric(
        direction: String = MORE_IS_BETTER,
    ): MetricsDirectoryEntity {
        return mockk {
            every { id } returns METRIC_ID
            every { name } returns METRIC_NAME
            every { unit } returns METRIC_UNIT
            every { this@mockk.direction } returns direction
        }
    }

    private fun metricType(
        agentType: String,
    ): InitiativeMetricTypeEntity {
        return mockk {
            every { this@mockk.agentType } returns agentType
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
            every { this@mockk.initiativeMetricType } returns
                initiativeMetricType
            every { metricDirectory } returns metric
            every { this@mockk.periodMonth } returns
                periodMonth.atDay(1)
            every { this@mockk.metricValue } returns metricValue
            every { this@mockk.targetValue } returns targetValue
        }
    }

    private companion object {
        val METRIC_ID: UUID =
            UUID.fromString(
                "a860b390-b739-48e6-a694-e96582eb4e95"
            )

        val PREVIOUS_MONTH: YearMonth =
            YearMonth.of(2026, 6)

        val BEFORE_PREVIOUS_MONTH: YearMonth =
            YearMonth.of(2026, 5)

        const val METRIC_NAME = "Точность"
        const val METRIC_UNIT = "percent"

        const val MORE_IS_BETTER =
            "more_is_better"

        const val LESS_IS_BETTER =
            "less_is_better"
    }
}
Тесты InitiativeMetricPreAnalyticsReader
@ExtendWith(MockKExtension::class)
class InitiativeMetricPreAnalyticsReaderTest {

    @MockK
    private lateinit var messageProvider:
        MessageProvider

    @MockK
    private lateinit var preAnalyticsProperties:
        PreAnalyticsProperties

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

    @BeforeEach
    fun setUp() {
        reader = InitiativeMetricPreAnalyticsReader(
            messageProvider = messageProvider,
            preAnalyticsProperties = preAnalyticsProperties,
            initiativeMetricTypeRepository =
                initiativeMetricTypeRepository,
            initiativeMetricValueRepository =
                initiativeMetricValueRepository,
            metricsDirectoryRepository =
                metricsDirectoryRepository,
            responseBuilder = responseBuilder,
        )
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

        every {
            messageProvider[
                INITIATIVE_METRIC_TYPES_NOT_FOUND
            ]
        } returns "Metric types not found for initiative {0}"

        assertThatThrownBy {
            reader.getPreAnalytics(
                initiativeId = INITIATIVE_ID,
                now = NOW,
            )
        }.isInstanceOf(AiConflictException::class.java)

        verify(exactly = 0) {
            metricsDirectoryRepository.findAllByNameIn(any())
            initiativeMetricValueRepository
                .findValuesForInitiativeMetrics(
                    any(),
                    any(),
                    any(),
                    any(),
                )
            responseBuilder.buildMetricsForAgentType(
                any(),
                any(),
                any(),
                any(),
                any(),
                any(),
            )
        }
    }

    @Test
    fun `should throw exception when configured metrics count is not four`() {
        every {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdAndAgentTypeIn(
                    initiativeId = INITIATIVE_ID,
                    agentTypes = SUPPORTED_AGENT_TYPES,
                )
        } returns listOf(
            metricType(AUTONOMOUS.value)
        )

        every {
            preAnalyticsProperties.metricNames
        } returns listOf(
            "Точность",
            "CSI",
            "Охват",
        )

        assertThatThrownBy {
            reader.getPreAnalytics(
                initiativeId = INITIATIVE_ID,
                now = NOW,
            )
        }
            .isInstanceOf(IllegalStateException::class.java)
            .hasMessage(
                "Exactly 4 unique pre-analytics metric names " +
                    "must be configured"
            )

        verify(exactly = 0) {
            metricsDirectoryRepository.findAllByNameIn(any())
        }
    }

    @Test
    fun `should throw exception when configured metric names contain duplicates`() {
        every {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdAndAgentTypeIn(
                    initiativeId = INITIATIVE_ID,
                    agentTypes = SUPPORTED_AGENT_TYPES,
                )
        } returns listOf(
            metricType(AUTONOMOUS.value)
        )

        every {
            preAnalyticsProperties.metricNames
        } returns listOf(
            "Точность",
            "CSI",
            "Охват",
            "Точность",
        )

        assertThatThrownBy {
            reader.getPreAnalytics(
                initiativeId = INITIATIVE_ID,
                now = NOW,
            )
        }
            .isInstanceOf(IllegalStateException::class.java)
            .hasMessage(
                "Exactly 4 unique pre-analytics metric names " +
                    "must be configured"
            )
    }

    @Test
    fun `should throw exception when configured metric is absent in directory`() {
        val configuredNames =
            listOf(
                "Точность",
                "CSI",
                "Охват",
                "Скорость",
            )

        val foundMetrics =
            configuredNames
                .dropLast(1)
                .mapIndexed { index, name ->
                    metric(
                        id = UUID.randomUUID(),
                        name = name,
                    )
                }

        mockReaderRepositories(
            configuredNames = configuredNames,
            metrics = foundMetrics,
        )

        assertThatThrownBy {
            reader.getPreAnalytics(
                initiativeId = INITIATIVE_ID,
                now = NOW,
            )
        }
            .isInstanceOf(IllegalStateException::class.java)
            .hasMessageContaining(
                "missing=[Скорость]"
            )

        verify(exactly = 0) {
            initiativeMetricValueRepository
                .findValuesForInitiativeMetrics(
                    any(),
                    any(),
                    any(),
                    any(),
                )
        }
    }

    @Test
    fun `should throw exception when directory contains duplicate metric`() {
        val configuredNames =
            listOf(
                "Точность",
                "CSI",
                "Охват",
                "Скорость",
            )

        val metrics =
            configuredNames.map { name ->
                metric(
                    id = UUID.randomUUID(),
                    name = name,
                )
            } + metric(
                id = UUID.randomUUID(),
                name = "Точность",
            )

        mockReaderRepositories(
            configuredNames = configuredNames,
            metrics = metrics,
        )

        assertThatThrownBy {
            reader.getPreAnalytics(
                initiativeId = INITIATIVE_ID,
                now = NOW,
            )
        }
            .isInstanceOf(IllegalStateException::class.java)
            .hasMessageContaining(
                "duplicated=[Точность]"
            )
    }

    @Test
    fun `should return response without error when at least one metric value is submitted`() {
        val configuredNames =
            listOf(
                "Точность",
                "CSI",
                "Охват",
                "Скорость",
            )

        val metrics =
            configuredNames.map { name ->
                metric(
                    id = UUID.randomUUID(),
                    name = name,
                )
            }

        val metricTypes =
            listOf(
                metricType(AUTONOMOUS.value),
                metricType(COPILOT.value),
            )

        val metricValues =
            listOf<InitiativeMetricValueEntity>(
                mockk()
            )

        every {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdAndAgentTypeIn(
                    initiativeId = INITIATIVE_ID,
                    agentTypes = SUPPORTED_AGENT_TYPES,
                )
        } returns metricTypes

        every {
            preAnalyticsProperties.metricNames
        } returns configuredNames

        every {
            metricsDirectoryRepository
                .findAllByNameIn(configuredNames)
        } returns metrics

        every {
            initiativeMetricValueRepository
                .findValuesForInitiativeMetrics(
                    initiativeId = INITIATIVE_ID,
                    agentTypes = SUPPORTED_AGENT_TYPES,
                    metricDirectoryIds =
                        metrics.map { it.id }.toSet(),
                    periodFrom =
                        BEFORE_PREVIOUS_MONTH.atDay(1),
                )
        } returns metricValues

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
                agentType = AUTONOMOUS.value,
                metricTypes = metricTypes,
                metrics = metrics,
                valuesByKey = any(),
                previousMonth = PREVIOUS_MONTH,
                beforePreviousMonth =
                    BEFORE_PREVIOUS_MONTH,
            )
        } returns listOf(autonomousItem)

        every {
            responseBuilder.buildMetricsForAgentType(
                agentType = COPILOT.value,
                metricTypes = metricTypes,
                metrics = metrics,
                valuesByKey = any(),
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

        assertThat(response.errorCode).isNull()

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
                    metricDirectoryIds =
                        metrics.map { it.id }.toSet(),
                    periodFrom =
                        LocalDate.of(2026, 5, 1),
                )
        }
    }

    @Test
    fun `should return error code when all metric values are absent`() {
        val configuredNames =
            listOf(
                "Точность",
                "CSI",
                "Охват",
                "Скорость",
            )

        val metrics =
            configuredNames.map { name ->
                metric(
                    id = UUID.randomUUID(),
                    name = name,
                )
            }

        val metricTypes =
            listOf(
                metricType(AUTONOMOUS.value)
            )

        every {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdAndAgentTypeIn(
                    initiativeId = INITIATIVE_ID,
                    agentTypes = SUPPORTED_AGENT_TYPES,
                )
        } returns metricTypes

        every {
            preAnalyticsProperties.metricNames
        } returns configuredNames

        every {
            metricsDirectoryRepository
                .findAllByNameIn(configuredNames)
        } returns metrics

        every {
            initiativeMetricValueRepository
                .findValuesForInitiativeMetrics(
                    initiativeId = INITIATIVE_ID,
                    agentTypes = SUPPORTED_AGENT_TYPES,
                    metricDirectoryIds =
                        metrics.map { it.id }.toSet(),
                    periodFrom =
                        BEFORE_PREVIOUS_MONTH.atDay(1),
                )
        } returns emptyList()

        val emptyItems =
            metrics.map { metric ->
                responseItem(
                    metric = metric,
                    value = null,
                )
            }

        every {
            responseBuilder.buildMetricsForAgentType(
                agentType = AUTONOMOUS.value,
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
                agentType = COPILOT.value,
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
            .isEqualTo(
                "Metric values have not been submitted"
            )

        assertThat(response.metricsAutonomous)
            .containsExactlyElementsOf(emptyItems)

        assertThat(response.metricsCopilot).isEmpty()
    }

    @Test
    fun `should preserve configured metrics order`() {
        val configuredNames =
            listOf(
                "Точность",
                "CSI",
                "Охват",
                "Скорость",
            )

        val metricsByConfiguredOrder =
            configuredNames.map { name ->
                metric(
                    id = UUID.randomUUID(),
                    name = name,
                )
            }

        val metricsFromRepository =
            metricsByConfiguredOrder.reversed()

        val metricTypes =
            listOf(
                metricType(AUTONOMOUS.value)
            )

        every {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdAndAgentTypeIn(
                    initiativeId = INITIATIVE_ID,
                    agentTypes = SUPPORTED_AGENT_TYPES,
                )
        } returns metricTypes

        every {
            preAnalyticsProperties.metricNames
        } returns configuredNames

        every {
            metricsDirectoryRepository
                .findAllByNameIn(configuredNames)
        } returns metricsFromRepository

        every {
            initiativeMetricValueRepository
                .findValuesForInitiativeMetrics(
                    any(),
                    any(),
                    any(),
                    any(),
                )
        } returns emptyList()

        every {
            responseBuilder.buildMetricsForAgentType(
                agentType = AUTONOMOUS.value,
                metricTypes = metricTypes,
                metrics = metricsByConfiguredOrder,
                valuesByKey = emptyMap(),
                previousMonth = PREVIOUS_MONTH,
                beforePreviousMonth =
                    BEFORE_PREVIOUS_MONTH,
            )
        } returns emptyList()

        every {
            responseBuilder.buildMetricsForAgentType(
                agentType = COPILOT.value,
                metricTypes = metricTypes,
                metrics = metricsByConfiguredOrder,
                valuesByKey = emptyMap(),
                previousMonth = PREVIOUS_MONTH,
                beforePreviousMonth =
                    BEFORE_PREVIOUS_MONTH,
            )
        } returns emptyList()

        reader.getPreAnalytics(
            initiativeId = INITIATIVE_ID,
            now = NOW,
        )

        verify(exactly = 2) {
            responseBuilder.buildMetricsForAgentType(
                agentType = any(),
                metricTypes = metricTypes,
                metrics = metricsByConfiguredOrder,
                valuesByKey = emptyMap(),
                previousMonth = PREVIOUS_MONTH,
                beforePreviousMonth =
                    BEFORE_PREVIOUS_MONTH,
            )
        }
    }

    private fun mockReaderRepositories(
        configuredNames: List<String>,
        metrics: List<MetricsDirectoryEntity>,
    ) {
        every {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdAndAgentTypeIn(
                    initiativeId = INITIATIVE_ID,
                    agentTypes = SUPPORTED_AGENT_TYPES,
                )
        } returns listOf(
            metricType(AUTONOMOUS.value)
        )

        every {
            preAnalyticsProperties.metricNames
        } returns configuredNames

        every {
            metricsDirectoryRepository
                .findAllByNameIn(configuredNames)
        } returns metrics
    }

    private fun metricType(
        agentType: String,
    ): InitiativeMetricTypeEntity {
        return mockk {
            every { this@mockk.agentType } returns agentType
        }
    }

    private fun metric(
        id: UUID,
        name: String,
    ): MetricsDirectoryEntity {
        return mockk {
            every { this@mockk.id } returns id
            every { this@mockk.name } returns name
            every { unit } returns "percent"
        }
    }

    private fun responseItem(
        metric: MetricsDirectoryEntity,
        value: BigDecimal?,
    ): InitiativeMetricPreAnalyticsItemResponse {
        return InitiativeMetricPreAnalyticsItemResponse(
            metricId = metric.id,
            name = metric.name,
            unit = metric.unit,
            value = value,
            targetValue =
                value?.let { BigDecimal("150") },
            deltaValue =
                value?.let { BigDecimal("20.00") },
        )
    }

    private companion object {
        const val INITIATIVE_ID = 100L

        val NOW: Instant =
            Instant.parse("2026-07-17T10:00:00Z")

        val PREVIOUS_MONTH: YearMonth =
            YearMonth.of(2026, 6)

        val BEFORE_PREVIOUS_MONTH: YearMonth =
            YearMonth.of(2026, 5)

        val SUPPORTED_AGENT_TYPES =
            setOf(
                AUTONOMOUS.value,
                COPILOT.value,
            )
    }
}
```
