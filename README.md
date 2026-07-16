```java
@ExtendWith(MockKExtension::class)
class InitiativeMetricValueReaderTest {

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
    private lateinit var metricResponseBuilder:
        InitiativeMetricResponseBuilder

    @InjectMockKs
    private lateinit var service: InitiativeMetricValueReader

    @Test
    fun `getInitiativeMetricValues should throw conflict when initiative metric types not found`() {
        // Given
        val initiativeId = 1L

        every {
            initiativeMetricTypeRepository.findAllByAiAgentId(
                initiativeId = initiativeId,
            )
        } returns emptyList()

        every {
            messageProvider[INITIATIVE_METRIC_TYPES_NOT_FOUND]
        } returns "Для инициативы с идентификатором {0} не найдены режимы работы"

        // When
        val exception = assertThrows<AiConflictException> {
            service.getInitiativeMetricValues(
                initiativeId = initiativeId,
            )
        }

        // Then
        assertEquals(
            "Для инициативы с идентификатором $initiativeId " +
                "не найдены режимы работы",
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
            initiativeMetricValueRepository.findValuesForInitiativeMetrics(
                initiativeId = any(),
                agentTypes = any(),
                metricDirectoryIds = any(),
                periodFrom = any(),
            )
        }

        verify(exactly = 0) {
            metricResponseBuilder.build(
                metrics = any(),
                requestedAgentTypes = any(),
                metricValues = any(),
                responseMonth = any(),
                clearRegularMetricValue = any(),
                includeCurrentPeriod = any(),
            )
        }
    }

    @Test
    fun `getInitiativeMetricValues should throw bad request when agent type is unknown`() {
        // Given
        val initiativeId = 1L

        val metricType = createMetricType(
            agentType = "wrong",
        )

        every {
            initiativeMetricTypeRepository.findAllByAiAgentId(
                initiativeId = initiativeId,
            )
        } returns listOf(metricType)

        every {
            messageProvider[WRONG_INITIATIVE_METRIC_AGENT_TYPE]
        } returns "Недопустимый режим работы инициативы: {0}"

        // When
        val exception = assertThrows<AiBadRequestException> {
            service.getInitiativeMetricValues(
                initiativeId = initiativeId,
            )
        }

        // Then
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
            initiativeMetricValueRepository.findValuesForInitiativeMetrics(
                initiativeId = any(),
                agentTypes = any(),
                metricDirectoryIds = any(),
                periodFrom = any(),
            )
        }
    }

    @Test
    fun `getInitiativeMetricValues should throw bad request when agent type is null`() {
        // Given
        val initiativeId = 1L

        val metricType = createMetricType(
            agentType = null,
        )

        every {
            initiativeMetricTypeRepository.findAllByAiAgentId(
                initiativeId = initiativeId,
            )
        } returns listOf(metricType)

        every {
            messageProvider[WRONG_INITIATIVE_METRIC_AGENT_TYPE]
        } returns "Недопустимый режим работы инициативы: {0}"

        // When
        val exception = assertThrows<AiBadRequestException> {
            service.getInitiativeMetricValues(
                initiativeId = initiativeId,
            )
        }

        // Then
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
            initiativeMetricValueRepository.findValuesForInitiativeMetrics(
                initiativeId = any(),
                agentTypes = any(),
                metricDirectoryIds = any(),
                periodFrom = any(),
            )
        }
    }

    @Test
    fun `getInitiativeMetricValues should return empty list when applicable metrics not found`() {
        // Given
        val initiativeId = 1L

        val metricTypes = listOf(
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

        // When
        val result = service.getInitiativeMetricValues(
            initiativeId = initiativeId,
        )

        // Then
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
            initiativeMetricValueRepository.findValuesForInitiativeMetrics(
                initiativeId = any(),
                agentTypes = any(),
                metricDirectoryIds = any(),
                periodFrom = any(),
            )
        }

        verify(exactly = 0) {
            metricResponseBuilder.historyPeriodFrom(any())
        }

        verify(exactly = 0) {
            metricResponseBuilder.build(
                metrics = any(),
                requestedAgentTypes = any(),
                metricValues = any(),
                responseMonth = any(),
                clearRegularMetricValue = any(),
                includeCurrentPeriod = any(),
            )
        }
    }

    @Test
    fun `getInitiativeMetricValues should use current month when current month values exist`() {
        // Given
        val initiativeId = 1L
        val metricId = UUID.randomUUID()
        val periodFrom = LocalDate.of(2025, 1, 1)

        val metricType = createMetricType(
            agentType = InitiativeMetricAgentType.AUTONOMOUS.value,
        )

        val metric = createMetric(
            metricId = metricId,
        )

        val metricValue = mockkMetricValue()

        val expectedResponse = listOf(
            mockkInitiativeMetricResponse(),
        )

        val periodMonthSlot = slot<LocalDate>()
        val historyMonthSlot = slot<YearMonth>()
        val responseMonthSlot = slot<YearMonth>()

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
            metricResponseBuilder.historyPeriodFrom(
                responseMonth = capture(historyMonthSlot),
            )
        } returns periodFrom

        every {
            initiativeMetricValueRepository.findValuesForInitiativeMetrics(
                initiativeId = initiativeId,
                agentTypes = setOf(
                    InitiativeMetricAgentType.AUTONOMOUS.value,
                ),
                metricDirectoryIds = setOf(metricId),
                periodFrom = periodFrom,
            )
        } returns listOf(metricValue)

        every {
            metricResponseBuilder.build(
                metrics = listOf(metric),
                requestedAgentTypes = setOf(
                    InitiativeMetricAgentType.AUTONOMOUS,
                ),
                metricValues = listOf(metricValue),
                responseMonth = capture(responseMonthSlot),
                clearRegularMetricValue = false,
                includeCurrentPeriod = false,
            )
        } returns expectedResponse

        // When
        val result = service.getInitiativeMetricValues(
            initiativeId = initiativeId,
        )

        // Then
        assertSame(
            expectedResponse,
            result,
        )

        val repositoryMonth =
            YearMonth.from(periodMonthSlot.captured)

        assertEquals(
            1,
            periodMonthSlot.captured.dayOfMonth,
        )

        assertEquals(
            repositoryMonth,
            responseMonthSlot.captured,
        )

        assertEquals(
            repositoryMonth,
            historyMonthSlot.captured,
        )

        verify(exactly = 1) {
            metricResponseBuilder.build(
                metrics = listOf(metric),
                requestedAgentTypes = setOf(
                    InitiativeMetricAgentType.AUTONOMOUS,
                ),
                metricValues = listOf(metricValue),
                responseMonth = responseMonthSlot.captured,
                clearRegularMetricValue = false,
                includeCurrentPeriod = false,
            )
        }
    }

    @Test
    fun `getInitiativeMetricValues should use current month when current month values do not exist`() {
        // Given
        val initiativeId = 1L
        val metricId = UUID.randomUUID()
        val periodFrom = LocalDate.of(2025, 1, 1)

        val metricType = createMetricType(
            agentType = InitiativeMetricAgentType.APPEALS.value,
        )

        val metric = createMetric(
            metricId = metricId,
        )

        val metricValue = mockkMetricValue()

        val expectedResponse = listOf(
            mockkInitiativeMetricResponse(),
        )

        val periodMonthSlot = slot<LocalDate>()
        val historyMonthSlot = slot<YearMonth>()
        val responseMonthSlot = slot<YearMonth>()

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
            metricResponseBuilder.historyPeriodFrom(
                responseMonth = capture(historyMonthSlot),
            )
        } returns periodFrom

        every {
            initiativeMetricValueRepository.findValuesForInitiativeMetrics(
                initiativeId = initiativeId,
                agentTypes = setOf(
                    InitiativeMetricAgentType.APPEALS.value,
                ),
                metricDirectoryIds = setOf(metricId),
                periodFrom = periodFrom,
            )
        } returns listOf(metricValue)

        every {
            metricResponseBuilder.build(
                metrics = listOf(metric),
                requestedAgentTypes = setOf(
                    InitiativeMetricAgentType.APPEALS,
                ),
                metricValues = listOf(metricValue),
                responseMonth = capture(responseMonthSlot),
                clearRegularMetricValue = true,
                includeCurrentPeriod = false,
            )
        } returns expectedResponse

        // When
        val result = service.getInitiativeMetricValues(
            initiativeId = initiativeId,
        )

        // Then
        assertSame(
            expectedResponse,
            result,
        )

        val repositoryMonth =
            YearMonth.from(periodMonthSlot.captured)

        assertEquals(
            1,
            periodMonthSlot.captured.dayOfMonth,
        )

        /*
         * Даже при отсутствии значений за текущий месяц
         * responseMonth должен оставаться текущим.
         *
         * Иначе предыдущий месяц получит индекс текущего периода
         * и будет удалён из periods при includeCurrentPeriod = false.
         */
        assertEquals(
            repositoryMonth,
            responseMonthSlot.captured,
        )

        assertEquals(
            repositoryMonth,
            historyMonthSlot.captured,
        )

        verify(exactly = 1) {
            metricResponseBuilder.build(
                metrics = listOf(metric),
                requestedAgentTypes = setOf(
                    InitiativeMetricAgentType.APPEALS,
                ),
                metricValues = listOf(metricValue),
                responseMonth = responseMonthSlot.captured,
                clearRegularMetricValue = true,
                includeCurrentPeriod = false,
            )
        }
    }

    @Test
    fun `getInitiativeMetricValues should pass copilot flag when only copilot is selected`() {
        // Given
        val initiativeId = 1L

        val metricType = createMetricType(
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

        // When
        val result = service.getInitiativeMetricValues(
            initiativeId = initiativeId,
        )

        // Then
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
        // Given
        val initiativeId = 1L
        val metricId = UUID.randomUUID()
        val periodFrom = LocalDate.of(2025, 1, 1)

        val firstMetricType = createMetricType(
            agentType = InitiativeMetricAgentType.COPILOT.value,
        )

        val secondMetricType = createMetricType(
            agentType = InitiativeMetricAgentType.COPILOT.value,
        )

        val metric = createMetric(
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
                    periodMonth = any(),
                )
        } returns true

        every {
            metricResponseBuilder.historyPeriodFrom(any())
        } returns periodFrom

        every {
            initiativeMetricValueRepository.findValuesForInitiativeMetrics(
                initiativeId = initiativeId,
                agentTypes = setOf(
                    InitiativeMetricAgentType.COPILOT.value,
                ),
                metricDirectoryIds = setOf(metricId),
                periodFrom = periodFrom,
            )
        } returns emptyList()

        every {
            metricResponseBuilder.build(
                metrics = listOf(metric),
                requestedAgentTypes = setOf(
                    InitiativeMetricAgentType.COPILOT,
                ),
                metricValues = emptyList(),
                responseMonth = any(),
                clearRegularMetricValue = false,
                includeCurrentPeriod = false,
            )
        } returns expectedResponse

        // When
        val result = service.getInitiativeMetricValues(
            initiativeId = initiativeId,
        )

        // Then
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
            initiativeMetricValueRepository.findValuesForInitiativeMetrics(
                initiativeId = initiativeId,
                agentTypes = setOf(
                    InitiativeMetricAgentType.COPILOT.value,
                ),
                metricDirectoryIds = setOf(metricId),
                periodFrom = periodFrom,
            )
        }

        verify(exactly = 1) {
            metricResponseBuilder.build(
                metrics = listOf(metric),
                requestedAgentTypes = setOf(
                    InitiativeMetricAgentType.COPILOT,
                ),
                metricValues = emptyList(),
                responseMonth = any(),
                clearRegularMetricValue = false,
                includeCurrentPeriod = false,
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

    private fun mockkMetricValue():
        InitiativeMetricValueEntity {
        return mockk()
    }

    private fun mockkInitiativeMetricResponse():
        InitiativeMetricResponse {
        return mockk()
    }
}
```
