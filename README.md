```java

@ExtendWith(MockKExtension::class)
class InitiativeMetricValueReaderTest {

    @MockK
    private lateinit var messageProvider: MessageProvider

    @MockK
    private lateinit var initiativeMetricTypeRepository: InitiativeMetricTypeRepository

    @MockK
    private lateinit var initiativeMetricValueRepository: InitiativeMetricValueRepository

    @MockK
    private lateinit var metricsDirectoryRepository: MetricsDirectoryRepository

    @MockK
    private lateinit var metricResponseBuilder: InitiativeMetricResponseBuilder

    @InjectMockKs
    private lateinit var service: InitiativeMetricValueReader

    @Test
    fun `getInitiativeMetricValues should throw bad request when agent type is unknown`() {
        val initiativeId = 1L
        val metricType = createMetricType(agentType = "wrong")

        every {
            initiativeMetricTypeRepository.findAllByAiAgentId(
                initiativeId = initiativeId,
            )
        } returns listOf(metricType)

        every {
            messageProvider[WRONG_INITIATIVE_METRIC_AGENT_TYPE]
        } returns "Недопустимый режим работы инициативы: {0}"

        val exception =
            assertThrows<AiBadRequestException> {
                service.getInitiativeMetricValues(
                    initiativeId = initiativeId,
                )
            }

        assertEquals(
            "Недопустимый режим работы инициативы: wrong",
            exception.message,
        )

        verify(exactly = 0) {
            metricsDirectoryRepository.findApplicableMetrics(
                autonomousSelected = any(),
                copilotSelected = any(),
                appealsSelected = any(),
            )
        }

        verify(exactly = 0) {
            initiativeMetricValueRepository
                .existsByInitiativeMetricTypeAiAgentIdAndPeriodMonth(
                    initiativeId = any(),
                    periodMonth = any(),
                )
        }

        verify(exactly = 0) {
            initiativeMetricValueRepository
                .findValuesForInitiativeMetricsInPeriodRange(
                    initiativeId = any(),
                    agentTypes = any(),
                    metricDirectoryIds = any(),
                    periodFrom = any(),
                    periodTo = any(),
                )
        }

        verify(exactly = 0) {
            metricResponseBuilder.build(
                metrics = any(),
                requestedAgentTypes = any(),
                metricValues = any(),
                reportingMonth = any(),
                clearRegularMetricValue = any(),
            )
        }
    }

    @Test
    fun `getInitiativeMetricValues should throw bad request when agent type is null`() {
        val initiativeId = 1L
        val metricType = createMetricType(agentType = null)

        every {
            initiativeMetricTypeRepository.findAllByAiAgentId(
                initiativeId = initiativeId,
            )
        } returns listOf(metricType)

        every {
            messageProvider[WRONG_INITIATIVE_METRIC_AGENT_TYPE]
        } returns "Недопустимый режим работы инициативы: {0}"

        val exception =
            assertThrows<AiBadRequestException> {
                service.getInitiativeMetricValues(
                    initiativeId = initiativeId,
                )
            }

        assertEquals(
            "Недопустимый режим работы инициативы: null",
            exception.message,
        )

        verify(exactly = 0) {
            metricsDirectoryRepository.findApplicableMetrics(
                autonomousSelected = any(),
                copilotSelected = any(),
                appealsSelected = any(),
            )
        }

        verify(exactly = 0) {
            initiativeMetricValueRepository
                .existsByInitiativeMetricTypeAiAgentIdAndPeriodMonth(
                    initiativeId = any(),
                    periodMonth = any(),
                )
        }

        verify(exactly = 0) {
            initiativeMetricValueRepository
                .findValuesForInitiativeMetricsInPeriodRange(
                    initiativeId = any(),
                    agentTypes = any(),
                    metricDirectoryIds = any(),
                    periodFrom = any(),
                    periodTo = any(),
                )
        }

        verify(exactly = 0) {
            metricResponseBuilder.build(
                metrics = any(),
                requestedAgentTypes = any(),
                metricValues = any(),
                reportingMonth = any(),
                clearRegularMetricValue = any(),
            )
        }
    }

    @Test
    fun `getInitiativeMetricValues should return empty list when metric types not found`() {
        val initiativeId = 1L

        every {
            initiativeMetricTypeRepository.findAllByAiAgentId(
                initiativeId = initiativeId,
            )
        } returns emptyList()

        val result =
            service.getInitiativeMetricValues(
                initiativeId = initiativeId,
            )

        assertTrue(result.isEmpty())

        verify(exactly = 0) {
            metricsDirectoryRepository.findApplicableMetrics(
                autonomousSelected = any(),
                copilotSelected = any(),
                appealsSelected = any(),
            )
        }

        verify(exactly = 0) {
            initiativeMetricValueRepository
                .existsByInitiativeMetricTypeAiAgentIdAndPeriodMonth(
                    initiativeId = any(),
                    periodMonth = any(),
                )
        }

        verify(exactly = 0) {
            initiativeMetricValueRepository
                .findValuesForInitiativeMetricsInPeriodRange(
                    initiativeId = any(),
                    agentTypes = any(),
                    metricDirectoryIds = any(),
                    periodFrom = any(),
                    periodTo = any(),
                )
        }

        verify(exactly = 0) {
            metricResponseBuilder.build(
                metrics = any(),
                requestedAgentTypes = any(),
                metricValues = any(),
                reportingMonth = any(),
                clearRegularMetricValue = any(),
            )
        }
    }

    @Test
    fun `getInitiativeMetricValues should return empty list when applicable metrics not found`() {
        val initiativeId = 1L

        val metricTypes =
            listOf(
                createMetricType(
                    agentType = InitiativeMetricAgentType.AUTONOMOUS.value,
                ),
                createMetricType(
                    agentType = InitiativeMetricAgentType.COPILOT.value,
                ),
                createMetricType(
                    agentType = InitiativeMetricAgentType.APPEALS.value,
                ),
            )

        every {
            initiativeMetricTypeRepository.findAllByAiAgentId(
                initiativeId = initiativeId,
            )
        } returns metricTypes

        every {
            metricsDirectoryRepository.findApplicableMetrics(
                autonomousSelected = true,
                copilotSelected = true,
                appealsSelected = true,
            )
        } returns emptyList()

        val result =
            service.getInitiativeMetricValues(
                initiativeId = initiativeId,
            )

        assertEquals(
            emptyList<InitiativeMetricResponse>(),
            result,
        )

        verify(exactly = 1) {
            metricsDirectoryRepository.findApplicableMetrics(
                autonomousSelected = true,
                copilotSelected = true,
                appealsSelected = true,
            )
        }

        verify(exactly = 0) {
            initiativeMetricValueRepository
                .existsByInitiativeMetricTypeAiAgentIdAndPeriodMonth(
                    initiativeId = any(),
                    periodMonth = any(),
                )
        }

        verify(exactly = 0) {
            initiativeMetricValueRepository
                .findValuesForInitiativeMetricsInPeriodRange(
                    initiativeId = any(),
                    agentTypes = any(),
                    metricDirectoryIds = any(),
                    periodFrom = any(),
                    periodTo = any(),
                )
        }

        verify(exactly = 0) {
            metricResponseBuilder.build(
                metrics = any(),
                requestedAgentTypes = any(),
                metricValues = any(),
                reportingMonth = any(),
                clearRegularMetricValue = any(),
            )
        }
    }

    @Test
    fun `getInitiativeMetricValues should use previous period and reporting month when reporting values exist`() {
        val initiativeId = 1L
        val metricId = UUID.randomUUID()

        val reportingMonth =
            YearMonth.now()
                .minusMonths(1)

        val previousPeriodMonth =
            reportingMonth.minusMonths(1)

        val metricType =
            createMetricType(
                agentType = InitiativeMetricAgentType.AUTONOMOUS.value,
            )

        val metric =
            createMetric(
                metricId = metricId,
            )

        val metricValue = mockkMetricValue()
        val expectedResponse =
            listOf(mockkInitiativeMetricResponse())

        every {
            metricValue.metricValue
        } returns BigDecimal.ONE

        every {
            metricValue.metricDirectory
        } returns metric

        every {
            expectedResponse.single().isActive
        } returns true

        val periodMonthSlot = slot<LocalDate>()
        val periodFromSlot = slot<LocalDate>()
        val periodToSlot = slot<LocalDate>()

        every {
            initiativeMetricTypeRepository.findAllByAiAgentId(
                initiativeId = initiativeId,
            )
        } returns listOf(metricType)

        every {
            metricsDirectoryRepository.findApplicableMetrics(
                autonomousSelected = true,
                copilotSelected = false,
                appealsSelected = false,
            )
        } returns listOf(metric)

        every {
            initiativeMetricValueRepository
                .existsByInitiativeMetricTypeAiAgentIdAndPeriodMonth(
                    initiativeId = initiativeId,
                    periodMonth = capture(periodMonthSlot),
                )
        } returns true

        every {
            initiativeMetricValueRepository
                .findValuesForInitiativeMetricsInPeriodRange(
                    initiativeId = initiativeId,
                    agentTypes =
                        setOf(
                            InitiativeMetricAgentType.AUTONOMOUS.value,
                        ),
                    metricDirectoryIds = setOf(metricId),
                    periodFrom = capture(periodFromSlot),
                    periodTo = capture(periodToSlot),
                )
        } returns listOf(metricValue)

        every {
            metricResponseBuilder.build(
                metrics = listOf(metric),
                requestedAgentTypes =
                    setOf(
                        InitiativeMetricAgentType.AUTONOMOUS,
                    ),
                metricValues = listOf(metricValue),
                reportingMonth = reportingMonth,
                clearRegularMetricValue = false,
            )
        } returns expectedResponse

        val result =
            service.getInitiativeMetricValues(
                initiativeId = initiativeId,
            )

        assertEquals(
            expectedResponse,
            result,
        )

        assertEquals(
            reportingMonth.atDay(1),
            periodMonthSlot.captured,
        )

        assertEquals(
            previousPeriodMonth.atDay(1),
            periodFromSlot.captured,
        )

        assertEquals(
            reportingMonth.atDay(1),
            periodToSlot.captured,
        )

        verify(exactly = 1) {
            metricResponseBuilder.build(
                metrics = listOf(metric),
                requestedAgentTypes =
                    setOf(
                        InitiativeMetricAgentType.AUTONOMOUS,
                    ),
                metricValues = listOf(metricValue),
                reportingMonth = reportingMonth,
                clearRegularMetricValue = false,
            )
        }
    }

    @Test
    fun `getInitiativeMetricValues should clear regular metric value when reporting month values do not exist`() {
        val initiativeId = 1L
        val metricId = UUID.randomUUID()

        val reportingMonth =
            YearMonth.now()
                .minusMonths(1)

        val previousPeriodMonth =
            reportingMonth.minusMonths(1)

        val metricType =
            createMetricType(
                agentType = InitiativeMetricAgentType.APPEALS.value,
            )

        val metric =
            createMetric(
                metricId = metricId,
            )

        val metricValue = mockkMetricValue()
        val expectedResponse =
            listOf(mockkInitiativeMetricResponse())

        every {
            metricValue.metricValue
        } returns BigDecimal.ONE

        every {
            metricValue.metricDirectory
        } returns metric

        every {
            expectedResponse.single().isActive
        } returns true

        val periodMonthSlot = slot<LocalDate>()
        val periodFromSlot = slot<LocalDate>()
        val periodToSlot = slot<LocalDate>()

        every {
            initiativeMetricTypeRepository.findAllByAiAgentId(
                initiativeId = initiativeId,
            )
        } returns listOf(metricType)

        every {
            metricsDirectoryRepository.findApplicableMetrics(
                autonomousSelected = false,
                copilotSelected = false,
                appealsSelected = true,
            )
        } returns listOf(metric)

        every {
            initiativeMetricValueRepository
                .existsByInitiativeMetricTypeAiAgentIdAndPeriodMonth(
                    initiativeId = initiativeId,
                    periodMonth = capture(periodMonthSlot),
                )
        } returns false

        every {
            initiativeMetricValueRepository
                .findValuesForInitiativeMetricsInPeriodRange(
                    initiativeId = initiativeId,
                    agentTypes =
                        setOf(
                            InitiativeMetricAgentType.APPEALS.value,
                        ),
                    metricDirectoryIds = setOf(metricId),
                    periodFrom = capture(periodFromSlot),
                    periodTo = capture(periodToSlot),
                )
        } returns listOf(metricValue)

        every {
            metricResponseBuilder.build(
                metrics = listOf(metric),
                requestedAgentTypes =
                    setOf(
                        InitiativeMetricAgentType.APPEALS,
                    ),
                metricValues = listOf(metricValue),
                reportingMonth = reportingMonth,
                clearRegularMetricValue = true,
            )
        } returns expectedResponse

        val result =
            service.getInitiativeMetricValues(
                initiativeId = initiativeId,
            )

        assertEquals(
            expectedResponse,
            result,
        )

        assertEquals(
            reportingMonth.atDay(1),
            periodMonthSlot.captured,
        )

        assertEquals(
            previousPeriodMonth.atDay(1),
            periodFromSlot.captured,
        )

        assertEquals(
            reportingMonth.atDay(1),
            periodToSlot.captured,
        )

        verify(exactly = 1) {
            metricResponseBuilder.build(
                metrics = listOf(metric),
                requestedAgentTypes =
                    setOf(
                        InitiativeMetricAgentType.APPEALS,
                    ),
                metricValues = listOf(metricValue),
                reportingMonth = reportingMonth,
                clearRegularMetricValue = true,
            )
        }
    }

    @Test
    fun `getInitiativeMetricValues should pass copilot flag when only copilot is selected`() {
        val initiativeId = 1L

        val metricType =
            createMetricType(
                agentType = InitiativeMetricAgentType.COPILOT.value,
            )

        every {
            initiativeMetricTypeRepository.findAllByAiAgentId(
                initiativeId = initiativeId,
            )
        } returns listOf(metricType)

        every {
            metricsDirectoryRepository.findApplicableMetrics(
                autonomousSelected = false,
                copilotSelected = true,
                appealsSelected = false,
            )
        } returns emptyList()

        val result =
            service.getInitiativeMetricValues(
                initiativeId = initiativeId,
            )

        assertEquals(
            emptyList<InitiativeMetricResponse>(),
            result,
        )

        verify(exactly = 1) {
            metricsDirectoryRepository.findApplicableMetrics(
                autonomousSelected = false,
                copilotSelected = true,
                appealsSelected = false,
            )
        }
    }

    @Test
    fun `getInitiativeMetricValues should remove duplicate agent types`() {
        val initiativeId = 1L
        val metricId = UUID.randomUUID()

        val reportingMonth =
            YearMonth.now()
                .minusMonths(1)

        val previousPeriodMonth =
            reportingMonth.minusMonths(1)

        val firstMetricType =
            createMetricType(
                agentType = InitiativeMetricAgentType.COPILOT.value,
            )

        val secondMetricType =
            createMetricType(
                agentType = InitiativeMetricAgentType.COPILOT.value,
            )

        val metric =
            createMetric(
                metricId = metricId,
            )

        val expectedResponse =
            emptyList<InitiativeMetricResponse>()

        every {
            initiativeMetricTypeRepository.findAllByAiAgentId(
                initiativeId = initiativeId,
            )
        } returns listOf(
            firstMetricType,
            secondMetricType,
        )

        every {
            metricsDirectoryRepository.findApplicableMetrics(
                autonomousSelected = false,
                copilotSelected = true,
                appealsSelected = false,
            )
        } returns listOf(metric)

        every {
            initiativeMetricValueRepository
                .existsByInitiativeMetricTypeAiAgentIdAndPeriodMonth(
                    initiativeId = initiativeId,
                    periodMonth = reportingMonth.atDay(1),
                )
        } returns true

        every {
            initiativeMetricValueRepository
                .findValuesForInitiativeMetricsInPeriodRange(
                    initiativeId = initiativeId,
                    agentTypes =
                        setOf(
                            InitiativeMetricAgentType.COPILOT.value,
                        ),
                    metricDirectoryIds = setOf(metricId),
                    periodFrom = previousPeriodMonth.atDay(1),
                    periodTo = reportingMonth.atDay(1),
                )
        } returns emptyList()

        every {
            metricResponseBuilder.build(
                metrics = listOf(metric),
                requestedAgentTypes =
                    setOf(
                        InitiativeMetricAgentType.COPILOT,
                    ),
                metricValues = emptyList(),
                reportingMonth = reportingMonth,
                clearRegularMetricValue = false,
            )
        } returns expectedResponse

        val result =
            service.getInitiativeMetricValues(
                initiativeId = initiativeId,
            )

        assertSame(
            expectedResponse,
            result,
        )

        verify(exactly = 1) {
            metricsDirectoryRepository.findApplicableMetrics(
                autonomousSelected = false,
                copilotSelected = true,
                appealsSelected = false,
            )
        }

        verify(exactly = 1) {
            initiativeMetricValueRepository
                .findValuesForInitiativeMetricsInPeriodRange(
                    initiativeId = initiativeId,
                    agentTypes =
                        setOf(
                            InitiativeMetricAgentType.COPILOT.value,
                        ),
                    metricDirectoryIds = setOf(metricId),
                    periodFrom = previousPeriodMonth.atDay(1),
                    periodTo = reportingMonth.atDay(1),
                )
        }

        verify(exactly = 1) {
            metricResponseBuilder.build(
                metrics = listOf(metric),
                requestedAgentTypes =
                    setOf(
                        InitiativeMetricAgentType.COPILOT,
                    ),
                metricValues = emptyList(),
                reportingMonth = reportingMonth,
                clearRegularMetricValue = false,
            )
        }
    }

    private fun createMetricType(
        agentType: String?,
    ): InitiativeMetricTypeEntity {
        val metricType =
            mockk<InitiativeMetricTypeEntity>()

        every {
            metricType.agentType
        } returns agentType

        return metricType
    }

    private fun createMetric(
        metricId: UUID,
    ): MetricsDirectoryEntity {
        val metric =
            mockk<MetricsDirectoryEntity>()

        every {
            metric.id
        } returns metricId

        return metric
    }

    private fun mockkMetricValue(): InitiativeMetricValueEntity {
        return mockk()
    }

    private fun mockkInitiativeMetricResponse(): InitiativeMetricResponse {
        return mockk()
    }
}

```
