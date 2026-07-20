```java
@ExtendWith(MockKExtension::class)
class InitiativeMetricPreAnalyticsReaderTest {

    @MockK
    private lateinit var messageProvider: MessageProvider

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
        reader =
            InitiativeMetricPreAnalyticsReader(
                messageProvider = messageProvider,
                preAnalyticsProperties =
                    preAnalyticsProperties,
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
        } returns
            "Metric types not found for initiative {0}"

        assertThatThrownBy {
            reader.getPreAnalytics(
                initiativeId = INITIATIVE_ID,
                now = NOW,
            )
        }.isInstanceOf(AiConflictException::class.java)

        verify(exactly = 0) {
            metricsDirectoryRepository.findAllById(
                any<Iterable<UUID>>(),
            )

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
                metricConfigurationsById = any(),
                valuesByKey = any(),
                previousMonth = any(),
                beforePreviousMonth = any(),
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
            metricType(
                InitiativeMetricAgentType
                    .AUTONOMOUS
                    .value,
            ),
        )

        every {
            preAnalyticsProperties.metrics
        } returns configuredMetrics().take(3)

        assertThatThrownBy {
            reader.getPreAnalytics(
                initiativeId = INITIATIVE_ID,
                now = NOW,
            )
        }
            .isInstanceOf(
                IllegalStateException::class.java,
            )
            .hasMessage(
                "Exactly 4 pre-analytics metrics " +
                    "must be configured",
            )

        verify(exactly = 0) {
            metricsDirectoryRepository.findAllById(
                any<Iterable<UUID>>(),
            )
        }
    }

    @Test
    fun `should throw exception when configured metric ids contain duplicates`() {
        val configurations =
            configuredMetrics().toMutableList()

        configurations[3] =
            configurations[3].copy(
                metricId = ACCURACY_METRIC_ID,
            )

        every {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdAndAgentTypeIn(
                    initiativeId = INITIATIVE_ID,
                    agentTypes = SUPPORTED_AGENT_TYPES,
                )
        } returns listOf(
            metricType(
                InitiativeMetricAgentType
                    .AUTONOMOUS
                    .value,
            ),
        )

        every {
            preAnalyticsProperties.metrics
        } returns configurations

        assertThatThrownBy {
            reader.getPreAnalytics(
                initiativeId = INITIATIVE_ID,
                now = NOW,
            )
        }
            .isInstanceOf(
                IllegalStateException::class.java,
            )
            .hasMessage(
                "Pre-analytics metricIds must be unique",
            )

        verify(exactly = 0) {
            metricsDirectoryRepository.findAllById(
                any<Iterable<UUID>>(),
            )
        }
    }

    @Test
    fun `should throw exception when configured metric codes contain duplicates`() {
        val configurations =
            configuredMetrics().toMutableList()

        configurations[3] =
            configurations[3].copy(
                code = ACCURACY_CODE,
            )

        every {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdAndAgentTypeIn(
                    initiativeId = INITIATIVE_ID,
                    agentTypes = SUPPORTED_AGENT_TYPES,
                )
        } returns listOf(
            metricType(
                InitiativeMetricAgentType
                    .AUTONOMOUS
                    .value,
            ),
        )

        every {
            preAnalyticsProperties.metrics
        } returns configurations

        assertThatThrownBy {
            reader.getPreAnalytics(
                initiativeId = INITIATIVE_ID,
                now = NOW,
            )
        }
            .isInstanceOf(
                IllegalStateException::class.java,
            )
            .hasMessage(
                "Pre-analytics metric codes must be unique",
            )

        verify(exactly = 0) {
            metricsDirectoryRepository.findAllById(
                any<Iterable<UUID>>(),
            )
        }
    }

    @Test
    fun `should throw exception when configured metric is absent in directory`() {
        val configurations =
            configuredMetrics()

        val foundMetrics =
            directoryMetrics().dropLast(1)

        mockReaderRepositories(
            configurations = configurations,
            metrics = foundMetrics,
        )

        assertThatThrownBy {
            reader.getPreAnalytics(
                initiativeId = INITIATIVE_ID,
                now = NOW,
            )
        }
            .isInstanceOf(
                IllegalStateException::class.java,
            )
            .hasMessageContaining(
                SPEED_METRIC_ID.toString(),
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
    fun `should return response without error when at least one metric value is submitted`() {
        val configurations =
            configuredMetrics()

        val metrics =
            directoryMetrics()

        val configuredIds =
            configurations.map { configuration ->
                requireNotNull(configuration.metricId)
            }

        val metricIds =
            configuredIds.toSet()

        val configurationsById =
            configurations.associateBy { configuration ->
                requireNotNull(configuration.metricId)
            }

        val metricTypes =
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

        every {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdAndAgentTypeIn(
                    initiativeId = INITIATIVE_ID,
                    agentTypes = SUPPORTED_AGENT_TYPES,
                )
        } returns metricTypes

        every {
            preAnalyticsProperties.metrics
        } returns configurations

        every {
            metricsDirectoryRepository.findAllById(
                configuredIds,
            )
        } returns metrics

        every {
            initiativeMetricValueRepository
                .findValuesForInitiativeMetrics(
                    initiativeId = INITIATIVE_ID,
                    agentTypes = SUPPORTED_AGENT_TYPES,
                    metricDirectoryIds = metricIds,
                    periodFrom =
                        BEFORE_PREVIOUS_MONTH.atDay(1),
                )
        } returns emptyList()

        val autonomousItem =
            responseItem(
                metric = metrics.first(),
                code = ACCURACY_CODE,
                value = BigDecimal("120"),
            )

        val copilotItem =
            responseItem(
                metric = metrics[1],
                code = CSI_CODE,
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
                metricConfigurationsById =
                    configurationsById,
                valuesByKey = any(),
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
                metricConfigurationsById =
                    configurationsById,
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
        val configurations =
            configuredMetrics()

        val metrics =
            directoryMetrics()

        val configuredIds =
            configurations.map { configuration ->
                requireNotNull(configuration.metricId)
            }

        val metricIds =
            configuredIds.toSet()

        val configurationsById =
            configurations.associateBy { configuration ->
                requireNotNull(configuration.metricId)
            }

        val metricTypes =
            listOf(
                metricType(
                    InitiativeMetricAgentType
                        .AUTONOMOUS
                        .value,
                ),
            )

        every {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdAndAgentTypeIn(
                    initiativeId = INITIATIVE_ID,
                    agentTypes = SUPPORTED_AGENT_TYPES,
                )
        } returns metricTypes

        every {
            preAnalyticsProperties.metrics
        } returns configurations

        every {
            metricsDirectoryRepository.findAllById(
                configuredIds,
            )
        } returns metrics

        every {
            initiativeMetricValueRepository
                .findValuesForInitiativeMetrics(
                    initiativeId = INITIATIVE_ID,
                    agentTypes = SUPPORTED_AGENT_TYPES,
                    metricDirectoryIds = metricIds,
                    periodFrom =
                        BEFORE_PREVIOUS_MONTH.atDay(1),
                )
        } returns emptyList()

        val emptyItems =
            metrics.mapIndexed { index, metric ->
                responseItem(
                    metric = metric,
                    code = configurations[index].code,
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
                metricConfigurationsById =
                    configurationsById,
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
                metricConfigurationsById =
                    configurationsById,
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
    fun `should preserve configured metrics order`() {
        val configurations =
            configuredMetrics()

        val metricsInConfiguredOrder =
            directoryMetrics()

        val metricsFromRepository =
            metricsInConfiguredOrder.reversed()

        val configuredIds =
            configurations.map { configuration ->
                requireNotNull(configuration.metricId)
            }

        val configurationsById =
            configurations.associateBy { configuration ->
                requireNotNull(configuration.metricId)
            }

        val metricTypes =
            listOf(
                metricType(
                    InitiativeMetricAgentType
                        .AUTONOMOUS
                        .value,
                ),
            )

        every {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdAndAgentTypeIn(
                    initiativeId = INITIATIVE_ID,
                    agentTypes = SUPPORTED_AGENT_TYPES,
                )
        } returns metricTypes

        every {
            preAnalyticsProperties.metrics
        } returns configurations

        every {
            metricsDirectoryRepository.findAllById(
                configuredIds,
            )
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
                agentType =
                    InitiativeMetricAgentType
                        .AUTONOMOUS
                        .value,
                metricTypes = metricTypes,
                metrics = metricsInConfiguredOrder,
                metricConfigurationsById =
                    configurationsById,
                valuesByKey = emptyMap(),
                previousMonth = PREVIOUS_MONTH,
                beforePreviousMonth =
                    BEFORE_PREVIOUS_MONTH,
            )
        } returns emptyList()

        every {
            responseBuilder.buildMetricsForAgentType(
                agentType =
                    InitiativeMetricAgentType
                        .COPILOT
                        .value,
                metricTypes = metricTypes,
                metrics = metricsInConfiguredOrder,
                metricConfigurationsById =
                    configurationsById,
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
                metrics = metricsInConfiguredOrder,
                metricConfigurationsById =
                    configurationsById,
                valuesByKey = emptyMap(),
                previousMonth = PREVIOUS_MONTH,
                beforePreviousMonth =
                    BEFORE_PREVIOUS_MONTH,
            )
        }
    }

    private fun mockReaderRepositories(
        configurations:
            List<PreAnalyticsMetricProperties>,
        metrics: List<MetricsDirectoryEntity>,
    ) {
        val configuredIds =
            configurations.map { configuration ->
                requireNotNull(configuration.metricId)
            }

        every {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdAndAgentTypeIn(
                    initiativeId = INITIATIVE_ID,
                    agentTypes = SUPPORTED_AGENT_TYPES,
                )
        } returns listOf(
            metricType(
                InitiativeMetricAgentType
                    .AUTONOMOUS
                    .value,
            ),
        )

        every {
            preAnalyticsProperties.metrics
        } returns configurations

        every {
            metricsDirectoryRepository.findAllById(
                configuredIds,
            )
        } returns metrics
    }

    private fun configuredMetrics():
        List<PreAnalyticsMetricProperties> {
        return listOf(
            PreAnalyticsMetricProperties(
                metricId = ACCURACY_METRIC_ID,
                code = ACCURACY_CODE,
            ),
            PreAnalyticsMetricProperties(
                metricId = CSI_METRIC_ID,
                code = CSI_CODE,
            ),
            PreAnalyticsMetricProperties(
                metricId = COVERAGE_METRIC_ID,
                code = COVERAGE_CODE,
            ),
            PreAnalyticsMetricProperties(
                metricId = SPEED_METRIC_ID,
                code = SPEED_CODE,
            ),
        )
    }

    private fun directoryMetrics():
        List<MetricsDirectoryEntity> {
        return listOf(
            metric(
                id = ACCURACY_METRIC_ID,
                name = "Полное название точности",
            ),
            metric(
                id = CSI_METRIC_ID,
                name = "Полное название CSI",
            ),
            metric(
                id = COVERAGE_METRIC_ID,
                name = "Полное название охвата",
            ),
            metric(
                id = SPEED_METRIC_ID,
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
        name: String,
    ): MetricsDirectoryEntity {
        return mockk {
            every {
                this@mockk.id
            } returns id

            every {
                this@mockk.name
            } returns name

            every {
                unit
            } returns "percent"
        }
    }

    private fun responseItem(
        metric: MetricsDirectoryEntity,
        code: String,
        value: BigDecimal?,
    ): InitiativeMetricPreAnalyticsItemResponse {
        return InitiativeMetricPreAnalyticsItemResponse(
            metricId = metric.id,
            code = code,
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

    private companion object {

        const val INITIATIVE_ID = 100L

        const val ACCURACY_CODE =
            "Точность"

        const val CSI_CODE =
            "Δ удовлетворённости"

        const val COVERAGE_CODE =
            "Охват пользователей"

        const val SPEED_CODE =
            "Скорость"

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
```
