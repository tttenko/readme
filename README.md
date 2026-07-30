```java
internal class CurrentPeriodMetricsDeviationFinderTest {

    @MockK
    private lateinit var initiativeMetricTypeRepository:
        InitiativeMetricTypeRepository

    @MockK
    private lateinit var metricsDirectoryRepository:
        MetricsDirectoryRepository

    @MockK
    private lateinit var initiativeMetricValueRepository:
        InitiativeMetricValueRepository

    @MockK
    private lateinit var metricApplicabilityPolicy:
        InitiativeMetricApplicabilityPolicy

    private lateinit var finder: CurrentPeriodMetricsDeviationFinder

    private val currentPeriod = LocalDate.of(2026, 7, 1)

    @BeforeEach
    fun setUp() {
        MockKAnnotations.init(this)

        finder = CurrentPeriodMetricsDeviationFinder(
            initiativeMetricTypeRepository =
                initiativeMetricTypeRepository,
            metricsDirectoryRepository =
                metricsDirectoryRepository,
            initiativeMetricValueRepository =
                initiativeMetricValueRepository,
            metricApplicabilityPolicy =
                metricApplicabilityPolicy,
        )
    }

    @Test
    fun `should return empty set and not call repositories when initiative ids are empty`() {
        val result =
            finder.findInitiativeIdsWithMissingMetrics(
                initiativeIds = emptySet(),
                currentPeriod = currentPeriod,
            )

        assertThat(result).isEmpty()

        verify(exactly = 0) {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdIn(any())
        }

        verify(exactly = 0) {
            metricsDirectoryRepository
                .findAllByActiveIsTrue()
        }

        verify(exactly = 0) {
            initiativeMetricValueRepository
                .findAllByInitiativeMetricTypeIdsAndPeriodMonth(
                    any(),
                    any(),
                )
        }
    }

    @Test
    fun `should return empty set when initiatives have no metric types`() {
        every {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdIn(setOf(1L))
        } returns emptyList()

        val result =
            finder.findInitiativeIdsWithMissingMetrics(
                initiativeIds = setOf(1L),
                currentPeriod = currentPeriod,
            )

        assertThat(result).isEmpty()

        verify(exactly = 1) {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdIn(setOf(1L))
        }

        verify(exactly = 0) {
            metricsDirectoryRepository.findAllByActiveIsTrue()
        }

        verify(exactly = 0) {
            initiativeMetricValueRepository
                .findAllByInitiativeMetricTypeIdsAndPeriodMonth(
                    any(),
                    any(),
                )
        }
    }

    @Test
    fun `should return empty set when there are no active metrics`() {
        val metricType =
            createMetricType(
                id = 10L,
                initiativeId = 1L,
                agentType =
                    InitiativeMetricAgentType.AUTONOMOUS.value,
            )

        every {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdIn(setOf(1L))
        } returns listOf(metricType)

        every {
            metricsDirectoryRepository.findAllByActiveIsTrue()
        } returns emptyList()

        val result =
            finder.findInitiativeIdsWithMissingMetrics(
                initiativeIds = setOf(1L),
                currentPeriod = currentPeriod,
            )

        assertThat(result).isEmpty()

        verify(exactly = 0) {
            initiativeMetricValueRepository
                .findAllByInitiativeMetricTypeIdsAndPeriodMonth(
                    any(),
                    any(),
                )
        }
    }

    @Test
    fun `should return initiative id when applicable metric value is missing`() {
        val metricType =
            createMetricType(
                id = 10L,
                initiativeId = 1L,
                agentType =
                    InitiativeMetricAgentType.AUTONOMOUS.value,
            )

        val metric = createMetric()

        every {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdIn(setOf(1L))
        } returns listOf(metricType)

        every {
            metricsDirectoryRepository.findAllByActiveIsTrue()
        } returns listOf(metric)

        every {
            metricApplicabilityPolicy.isApplicable(
                metric = metric,
                agentType =
                    InitiativeMetricAgentType.AUTONOMOUS,
            )
        } returns true

        every {
            initiativeMetricValueRepository
                .findAllByInitiativeMetricTypeIdsAndPeriodMonth(
                    initiativeMetricTypeIds = setOf(10L),
                    periodMonth = currentPeriod,
                )
        } returns emptyList()

        val result =
            finder.findInitiativeIdsWithMissingMetrics(
                initiativeIds = setOf(1L),
                currentPeriod = currentPeriod,
            )

        assertThat(result).containsExactly(1L)
    }

    @Test
    fun `should not return initiative id when all applicable metrics are filled`() {
        val metricType =
            createMetricType(
                id = 10L,
                initiativeId = 1L,
                agentType =
                    InitiativeMetricAgentType.AUTONOMOUS.value,
            )

        val metricId = UUID.randomUUID()
        val metric = createMetric(id = metricId)

        val metricValue =
            createMetricValue(
                metricType = metricType,
                metric = metric,
                value = BigDecimal.TEN,
            )

        every {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdIn(setOf(1L))
        } returns listOf(metricType)

        every {
            metricsDirectoryRepository.findAllByActiveIsTrue()
        } returns listOf(metric)

        every {
            metricApplicabilityPolicy.isApplicable(
                metric = metric,
                agentType =
                    InitiativeMetricAgentType.AUTONOMOUS,
            )
        } returns true

        every {
            initiativeMetricValueRepository
                .findAllByInitiativeMetricTypeIdsAndPeriodMonth(
                    initiativeMetricTypeIds = setOf(10L),
                    periodMonth = currentPeriod,
                )
        } returns listOf(metricValue)

        val result =
            finder.findInitiativeIdsWithMissingMetrics(
                initiativeIds = setOf(1L),
                currentPeriod = currentPeriod,
            )

        assertThat(result).isEmpty()
    }

    @Test
    fun `should return initiative id when metric value entity exists but metricValue is null`() {
        val metricType =
            createMetricType(
                id = 10L,
                initiativeId = 1L,
                agentType =
                    InitiativeMetricAgentType.AUTONOMOUS.value,
            )

        val metric = createMetric()

        val metricValue =
            createMetricValue(
                metricType = metricType,
                metric = metric,
                value = null,
            )

        every {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdIn(setOf(1L))
        } returns listOf(metricType)

        every {
            metricsDirectoryRepository.findAllByActiveIsTrue()
        } returns listOf(metric)

        every {
            metricApplicabilityPolicy.isApplicable(
                metric,
                InitiativeMetricAgentType.AUTONOMOUS,
            )
        } returns true

        every {
            initiativeMetricValueRepository
                .findAllByInitiativeMetricTypeIdsAndPeriodMonth(
                    setOf(10L),
                    currentPeriod,
                )
        } returns listOf(metricValue)

        val result =
            finder.findInitiativeIdsWithMissingMetrics(
                initiativeIds = setOf(1L),
                currentPeriod = currentPeriod,
            )

        assertThat(result).containsExactly(1L)
    }

    @Test
    fun `should ignore metrics that are not applicable to initiative agent type`() {
        val metricType =
            createMetricType(
                id = 10L,
                initiativeId = 1L,
                agentType =
                    InitiativeMetricAgentType.COPILOT.value,
            )

        val metric = createMetric()

        every {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdIn(setOf(1L))
        } returns listOf(metricType)

        every {
            metricsDirectoryRepository.findAllByActiveIsTrue()
        } returns listOf(metric)

        every {
            metricApplicabilityPolicy.isApplicable(
                metric = metric,
                agentType =
                    InitiativeMetricAgentType.COPILOT,
            )
        } returns false

        every {
            initiativeMetricValueRepository
                .findAllByInitiativeMetricTypeIdsAndPeriodMonth(
                    setOf(10L),
                    currentPeriod,
                )
        } returns emptyList()

        val result =
            finder.findInitiativeIdsWithMissingMetrics(
                initiativeIds = setOf(1L),
                currentPeriod = currentPeriod,
            )

        assertThat(result).isEmpty()
    }

    @Test
    fun `should ignore metric type with unknown agent type`() {
        val metricType =
            createMetricType(
                id = 10L,
                initiativeId = 1L,
                agentType = "unknown",
            )

        val metric = createMetric()

        every {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdIn(setOf(1L))
        } returns listOf(metricType)

        every {
            metricsDirectoryRepository.findAllByActiveIsTrue()
        } returns listOf(metric)

        every {
            initiativeMetricValueRepository
                .findAllByInitiativeMetricTypeIdsAndPeriodMonth(
                    setOf(10L),
                    currentPeriod,
                )
        } returns emptyList()

        val result =
            finder.findInitiativeIdsWithMissingMetrics(
                initiativeIds = setOf(1L),
                currentPeriod = currentPeriod,
            )

        assertThat(result).isEmpty()

        verify(exactly = 0) {
            metricApplicabilityPolicy.isApplicable(
                any(),
                any(),
            )
        }
    }

    @Test
    fun `should ignore metric type without initiative`() {
        val metricType =
            createMetricType(
                id = 10L,
                initiativeId = null,
                agentType =
                    InitiativeMetricAgentType.AUTONOMOUS.value,
            )

        val metric = createMetric()

        every {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdIn(setOf(1L))
        } returns listOf(metricType)

        every {
            metricsDirectoryRepository.findAllByActiveIsTrue()
        } returns listOf(metric)

        every {
            initiativeMetricValueRepository
                .findAllByInitiativeMetricTypeIdsAndPeriodMonth(
                    setOf(10L),
                    currentPeriod,
                )
        } returns emptyList()

        val result =
            finder.findInitiativeIdsWithMissingMetrics(
                initiativeIds = setOf(1L),
                currentPeriod = currentPeriod,
            )

        assertThat(result).isEmpty()

        verify(exactly = 0) {
            metricApplicabilityPolicy.isApplicable(
                any(),
                any(),
            )
        }
    }

    @Test
    fun `should return only initiative with missing metrics when another initiative is fully filled`() {
        val firstMetricType =
            createMetricType(
                id = 10L,
                initiativeId = 1L,
                agentType =
                    InitiativeMetricAgentType.AUTONOMOUS.value,
            )

        val secondMetricType =
            createMetricType(
                id = 20L,
                initiativeId = 2L,
                agentType =
                    InitiativeMetricAgentType.AUTONOMOUS.value,
            )

        val metric = createMetric()

        val secondInitiativeValue =
            createMetricValue(
                metricType = secondMetricType,
                metric = metric,
                value = BigDecimal.ONE,
            )

        every {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdIn(setOf(1L, 2L))
        } returns listOf(
            firstMetricType,
            secondMetricType,
        )

        every {
            metricsDirectoryRepository.findAllByActiveIsTrue()
        } returns listOf(metric)

        every {
            metricApplicabilityPolicy.isApplicable(
                metric = metric,
                agentType =
                    InitiativeMetricAgentType.AUTONOMOUS,
            )
        } returns true

        every {
            initiativeMetricValueRepository
                .findAllByInitiativeMetricTypeIdsAndPeriodMonth(
                    initiativeMetricTypeIds = setOf(10L, 20L),
                    periodMonth = currentPeriod,
                )
        } returns listOf(secondInitiativeValue)

        val result =
            finder.findInitiativeIdsWithMissingMetrics(
                initiativeIds = setOf(1L, 2L),
                currentPeriod = currentPeriod,
            )

        assertThat(result).containsExactly(1L)
    }

    @Test
    fun `should require values for all applicable metrics`() {
        val metricType =
            createMetricType(
                id = 10L,
                initiativeId = 1L,
                agentType =
                    InitiativeMetricAgentType.AUTONOMOUS.value,
            )

        val firstMetric = createMetric()
        val secondMetric = createMetric()

        val firstMetricValue =
            createMetricValue(
                metricType = metricType,
                metric = firstMetric,
                value = BigDecimal.ONE,
            )

        every {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdIn(setOf(1L))
        } returns listOf(metricType)

        every {
            metricsDirectoryRepository.findAllByActiveIsTrue()
        } returns listOf(
            firstMetric,
            secondMetric,
        )

        every {
            metricApplicabilityPolicy.isApplicable(
                metric = firstMetric,
                agentType =
                    InitiativeMetricAgentType.AUTONOMOUS,
            )
        } returns true

        every {
            metricApplicabilityPolicy.isApplicable(
                metric = secondMetric,
                agentType =
                    InitiativeMetricAgentType.AUTONOMOUS,
            )
        } returns true

        every {
            initiativeMetricValueRepository
                .findAllByInitiativeMetricTypeIdsAndPeriodMonth(
                    setOf(10L),
                    currentPeriod,
                )
        } returns listOf(firstMetricValue)

        val result =
            finder.findInitiativeIdsWithMissingMetrics(
                initiativeIds = setOf(1L),
                currentPeriod = currentPeriod,
            )

        assertThat(result).containsExactly(1L)
    }

    @Test
    fun `should require values for every initiative agent type`() {
        val autonomousMetricType =
            createMetricType(
                id = 10L,
                initiativeId = 1L,
                agentType =
                    InitiativeMetricAgentType.AUTONOMOUS.value,
            )

        val copilotMetricType =
            createMetricType(
                id = 20L,
                initiativeId = 1L,
                agentType =
                    InitiativeMetricAgentType.COPILOT.value,
            )

        val metric = createMetric()

        val autonomousValue =
            createMetricValue(
                metricType = autonomousMetricType,
                metric = metric,
                value = BigDecimal.ONE,
            )

        every {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdIn(setOf(1L))
        } returns listOf(
            autonomousMetricType,
            copilotMetricType,
        )

        every {
            metricsDirectoryRepository.findAllByActiveIsTrue()
        } returns listOf(metric)

        every {
            metricApplicabilityPolicy.isApplicable(
                metric,
                InitiativeMetricAgentType.AUTONOMOUS,
            )
        } returns true

        every {
            metricApplicabilityPolicy.isApplicable(
                metric,
                InitiativeMetricAgentType.COPILOT,
            )
        } returns true

        every {
            initiativeMetricValueRepository
                .findAllByInitiativeMetricTypeIdsAndPeriodMonth(
                    setOf(10L, 20L),
                    currentPeriod,
                )
        } returns listOf(autonomousValue)

        val result =
            finder.findInitiativeIdsWithMissingMetrics(
                initiativeIds = setOf(1L),
                currentPeriod = currentPeriod,
            )

        assertThat(result).containsExactly(1L)
    }

    @Test
    fun `should ignore filled metric value without metric type reference`() {
        val metricType =
            createMetricType(
                id = 10L,
                initiativeId = 1L,
                agentType =
                    InitiativeMetricAgentType.AUTONOMOUS.value,
            )

        val metric = createMetric()

        val invalidMetricValue =
            createMetricValue(
                metricType = null,
                metric = metric,
                value = BigDecimal.ONE,
            )

        every {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdIn(setOf(1L))
        } returns listOf(metricType)

        every {
            metricsDirectoryRepository.findAllByActiveIsTrue()
        } returns listOf(metric)

        every {
            metricApplicabilityPolicy.isApplicable(
                metric,
                InitiativeMetricAgentType.AUTONOMOUS,
            )
        } returns true

        every {
            initiativeMetricValueRepository
                .findAllByInitiativeMetricTypeIdsAndPeriodMonth(
                    setOf(10L),
                    currentPeriod,
                )
        } returns listOf(invalidMetricValue)

        val result =
            finder.findInitiativeIdsWithMissingMetrics(
                initiativeIds = setOf(1L),
                currentPeriod = currentPeriod,
            )

        assertThat(result).containsExactly(1L)
    }

    @Test
    fun `should ignore filled metric value without metric directory reference`() {
        val metricType =
            createMetricType(
                id = 10L,
                initiativeId = 1L,
                agentType =
                    InitiativeMetricAgentType.AUTONOMOUS.value,
            )

        val metric = createMetric()

        val invalidMetricValue =
            createMetricValue(
                metricType = metricType,
                metric = null,
                value = BigDecimal.ONE,
            )

        every {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdIn(setOf(1L))
        } returns listOf(metricType)

        every {
            metricsDirectoryRepository.findAllByActiveIsTrue()
        } returns listOf(metric)

        every {
            metricApplicabilityPolicy.isApplicable(
                metric,
                InitiativeMetricAgentType.AUTONOMOUS,
            )
        } returns true

        every {
            initiativeMetricValueRepository
                .findAllByInitiativeMetricTypeIdsAndPeriodMonth(
                    setOf(10L),
                    currentPeriod,
                )
        } returns listOf(invalidMetricValue)

        val result =
            finder.findInitiativeIdsWithMissingMetrics(
                initiativeIds = setOf(1L),
                currentPeriod = currentPeriod,
            )

        assertThat(result).containsExactly(1L)
    }

    private fun createMetricType(
        id: Long,
        initiativeId: Long?,
        agentType: String?,
    ): InitiativeMetricTypeEntity {
        val metricType = mockk<InitiativeMetricTypeEntity>()
        val agent =
            initiativeId?.let {
                mockk<AIAgentEntity>().also { agent ->
                    every { agent.id } returns initiativeId
                }
            }

        every { metricType.id } returns id
        every { metricType.aiAgent } returns agent
        every { metricType.agentType } returns agentType

        return metricType
    }

    private fun createMetric(
        id: UUID = UUID.randomUUID(),
    ): MetricsDirectoryEntity {
        val metric = mockk<MetricsDirectoryEntity>()

        every { metric.id } returns id

        return metric
    }

    private fun createMetricValue(
        metricType: InitiativeMetricTypeEntity?,
        metric: MetricsDirectoryEntity?,
        value: BigDecimal?,
    ): InitiativeMetricValueEntity {
        val metricValue = mockk<InitiativeMetricValueEntity>()

        every { metricValue.initiativeMetricType } returns metricType
        every { metricValue.metricDirectory } returns metric
        every { metricValue.metricValue } returns value

        return metricValue
    }
}
```
