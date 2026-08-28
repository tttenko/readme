```java

@ExtendWith(MockKExtension::class)
class InitiativeAgentTypesUpdaterTest {

    @MockK
    private lateinit var messageProvider: MessageProvider

    @MockK
    private lateinit var aiAgentRepository: AIAgentRepository

    @MockK
    private lateinit var initiativeMetricTypeRepository: InitiativeMetricTypeRepository

    @MockK
    private lateinit var initiativeMetricValueRepository: InitiativeMetricValueRepository

    @MockK
    private lateinit var metricsDirectoryRepository: MetricsDirectoryRepository

    @MockK
    private lateinit var permissionValidator: InitiativeAgentTypesPermissionValidator

    @MockK
    private lateinit var metricResponseBuilder: InitiativeMetricResponseBuilder

    private lateinit var updater: InitiativeAgentTypesUpdater

    @BeforeEach
    fun setUp() {
        updater = InitiativeAgentTypesUpdater(
            messageProvider = messageProvider,
            aiAgentRepository = aiAgentRepository,
            initiativeMetricTypeRepository = initiativeMetricTypeRepository,
            initiativeMetricValueRepository = initiativeMetricValueRepository,
            metricsDirectoryRepository = metricsDirectoryRepository,
            permissionValidator = permissionValidator,
            metricResponseBuilder = metricResponseBuilder,
        )
    }

    @Test
    fun `update should throw bad request when initiative not found`() {
        // Given
        val initiativeId = 1L

        every { aiAgentRepository.findByIdOrNull(id = initiativeId) } returns null
        every { messageProvider[INITIATIVE_NOT_FOUND] } returns "Initiative {0} not found"

        val request = UpdateInitiativeAgentTypesRequest(agentTypes = setOf("copilot"))

        // When & Then
        assertThrows<AiBadRequestException> {
            updater.updateInitiativeAgentType(initiativeId = initiativeId, request = request)
        }

        verify(exactly = 0) { permissionValidator.validate(any()) }
        verify(exactly = 0) { initiativeMetricTypeRepository.findAllByAiAgentId(any()) }
    }

    @Test
    fun `update should throw bad request when agent type is wrong`() {
        // Given
        val initiativeId = 1L
        val initiative = createInitiative(id = initiativeId)

        every { aiAgentRepository.findByIdOrNull(id = initiativeId) } returns initiative
        every { permissionValidator.validate(initiative = initiative) } just Runs
        every { messageProvider[WRONG_INITIATIVE_METRIC_AGENT_TYPE] } returns "Wrong agent type {0}"

        val request = UpdateInitiativeAgentTypesRequest(agentTypes = setOf("wrong"))

        // When & Then
        assertThrows<AiBadRequestException> {
            updater.updateInitiativeAgentType(initiativeId = initiativeId, request = request)
        }

        verify(exactly = 1) { permissionValidator.validate(initiative = initiative) }
        verify(exactly = 0) { initiativeMetricTypeRepository.findAllByAiAgentId(any()) }

        verify(exactly = 0) {
            metricsDirectoryRepository.findApplicableMetrics(
                autonomousSelected = any(),
                copilotSelected = any(),
                appealsSelected = any(),
            )
        }
    }

    @Test
    fun `update should create new metric types and return built response`() {
        // Given
        val initiativeId = 1L
        val initiative = createInitiative(id = initiativeId)
        val metricId = UUID.randomUUID()

        val reportingMonth = YearMonth.now().minusMonths(1)
        val previousPeriodMonth = reportingMonth.minusMonths(1)

        val metric = createMetric(
            id = metricId,
            copilotApplicability = true,
        )

        val metricValue = InitiativeMetricValueEntity(
            initiativeMetricType = InitiativeMetricTypeEntity(
                aiAgent = initiative,
                agentType = "copilot",
            ).apply {
                id = 10L
            },
            metricDirectory = metric,
            periodMonth = reportingMonth.atDay(1),
            metricValue = BigDecimal.TEN,
            targetValue = BigDecimal.valueOf(20),
        ).apply {
            id = 100L
        }

        val expectedResponse = listOf(
            InitiativeMetricResponse(
                id = metricId,
                name = "Метрика",
                unit = "шт",
                direction = "growth",
                agentType = "copilot",
                isActive = true,
                description = "Описание",
                frequency = "monthly",
                metricValue = BigDecimal.TEN,
                targetValue = BigDecimal.valueOf(20),
                planExecution = null,
                periods = emptyList(),
            )
        )

        every { aiAgentRepository.findByIdOrNull(id = initiativeId) } returns initiative
        every { permissionValidator.validate(initiative = initiative) } just Runs

        every {
            initiativeMetricTypeRepository.findAllByAiAgentId(initiativeId = initiativeId)
        } returns emptyList()

        every {
            initiativeMetricTypeRepository.saveAll(any<List<InitiativeMetricTypeEntity>>())
        } answers {
            firstArg<List<InitiativeMetricTypeEntity>>()
        }

        every {
            metricsDirectoryRepository.findApplicableMetrics(
                autonomousSelected = false,
                copilotSelected = true,
                appealsSelected = false,
            )
        } returns listOf(metric)

        every {
            initiativeMetricValueRepository.findValuesForInitiativeMetricsInPeriodRange(
                initiativeId = initiativeId,
                agentTypes = setOf("copilot"),
                metricDirectoryIds = setOf(metricId),
                periodFrom = previousPeriodMonth.atDay(1),
                periodTo = reportingMonth.atDay(1),
            )
        } returns listOf(metricValue)

        every {
            metricResponseBuilder.build(
                metrics = listOf(metric),
                requestedAgentTypes = setOf(InitiativeMetricAgentType.COPILOT),
                metricValues = listOf(metricValue),
                reportingMonth = reportingMonth,
            )
        } returns expectedResponse

        val request = UpdateInitiativeAgentTypesRequest(agentTypes = setOf("copilot"))

        // When
        val result = updater.updateInitiativeAgentType(initiativeId = initiativeId, request = request)

        // Then
        assertEquals(expectedResponse, result)

        verify(exactly = 1) { permissionValidator.validate(initiative = initiative) }

        verify(exactly = 1) {
            initiativeMetricTypeRepository.saveAll(
                match<List<InitiativeMetricTypeEntity>> { entities ->
                    entities.size == 1 &&
                        entities.first().aiAgent == initiative &&
                        entities.first().agentType == "copilot"
                }
            )
        }

        verify(exactly = 0) {
            initiativeMetricValueRepository.deleteAllByInitiativeMetricTypeIds(any())
        }

        verify(exactly = 1) {
            initiativeMetricValueRepository.findValuesForInitiativeMetricsInPeriodRange(
                initiativeId = initiativeId,
                agentTypes = setOf("copilot"),
                metricDirectoryIds = setOf(metricId),
                periodFrom = previousPeriodMonth.atDay(1),
                periodTo = reportingMonth.atDay(1),
            )
        }

        verify(exactly = 1) {
            metricResponseBuilder.build(
                metrics = listOf(metric),
                requestedAgentTypes = setOf(InitiativeMetricAgentType.COPILOT),
                metricValues = listOf(metricValue),
                reportingMonth = reportingMonth,
            )
        }
    }

    @Test
    fun `update should delete removed metric types and their values`() {
        // Given
        val initiativeId = 1L
        val initiative = createInitiative(id = initiativeId)

        val reportingMonth = YearMonth.now().minusMonths(1)
        val previousPeriodMonth = reportingMonth.minusMonths(1)

        val existingCopilot = InitiativeMetricTypeEntity(
            aiAgent = initiative,
            agentType = "copilot",
        ).apply {
            id = 10L
        }

        val existingAutonomous = InitiativeMetricTypeEntity(
            aiAgent = initiative,
            agentType = "autonomous",
        ).apply {
            id = 20L
        }

        val autonomousMetric = createMetric(
            id = UUID.randomUUID(),
            autonomousApplicability = true,
        )

        every { aiAgentRepository.findByIdOrNull(id = initiativeId) } returns initiative
        every { permissionValidator.validate(initiative = initiative) } just Runs

        every {
            initiativeMetricTypeRepository.findAllByAiAgentId(initiativeId = initiativeId)
        } returns listOf(existingCopilot, existingAutonomous)

        every {
            initiativeMetricValueRepository.deleteAllByInitiativeMetricTypeIds(
                initiativeMetricTypeIds = setOf(10L),
            )
        } just Runs

        every {
            initiativeMetricTypeRepository.deleteAll(listOf(existingCopilot))
        } just Runs

        every {
            metricsDirectoryRepository.findApplicableMetrics(
                autonomousSelected = true,
                copilotSelected = false,
                appealsSelected = false,
            )
        } returns listOf(autonomousMetric)

        every {
            initiativeMetricValueRepository.findValuesForInitiativeMetricsInPeriodRange(
                initiativeId = initiativeId,
                agentTypes = setOf("autonomous"),
                metricDirectoryIds = setOf(autonomousMetric.id),
                periodFrom = previousPeriodMonth.atDay(1),
                periodTo = reportingMonth.atDay(1),
            )
        } returns emptyList()

        every {
            metricResponseBuilder.build(
                metrics = listOf(autonomousMetric),
                requestedAgentTypes = setOf(InitiativeMetricAgentType.AUTONOMOUS),
                metricValues = emptyList(),
                reportingMonth = reportingMonth,
            )
        } returns emptyList()

        val request = UpdateInitiativeAgentTypesRequest(agentTypes = setOf("autonomous"))

        // When
        val result = updater.updateInitiativeAgentType(initiativeId = initiativeId, request = request)

        // Then
        assertTrue(result.isEmpty())

        verify(exactly = 1) {
            initiativeMetricValueRepository.deleteAllByInitiativeMetricTypeIds(
                initiativeMetricTypeIds = setOf(10L),
            )
        }

        verify(exactly = 1) {
            initiativeMetricTypeRepository.deleteAll(listOf(existingCopilot))
        }

        verify(exactly = 0) {
            initiativeMetricTypeRepository.saveAll(any<List<InitiativeMetricTypeEntity>>())
        }

        verify(exactly = 1) {
            initiativeMetricValueRepository.findValuesForInitiativeMetricsInPeriodRange(
                initiativeId = initiativeId,
                agentTypes = setOf("autonomous"),
                metricDirectoryIds = setOf(autonomousMetric.id),
                periodFrom = previousPeriodMonth.atDay(1),
                periodTo = reportingMonth.atDay(1),
            )
        }

        verify(exactly = 1) {
            metricResponseBuilder.build(
                metrics = listOf(autonomousMetric),
                requestedAgentTypes = setOf(InitiativeMetricAgentType.AUTONOMOUS),
                metricValues = emptyList(),
                reportingMonth = reportingMonth,
            )
        }
    }

    @Test
    fun `update should return empty list when no applicable metrics found`() {
        // Given
        val initiativeId = 1L
        val initiative = createInitiative(id = initiativeId)

        every { aiAgentRepository.findByIdOrNull(id = initiativeId) } returns initiative
        every { permissionValidator.validate(initiative = initiative) } just Runs

        every {
            initiativeMetricTypeRepository.findAllByAiAgentId(initiativeId = initiativeId)
        } returns emptyList()

        every {
            initiativeMetricTypeRepository.saveAll(any<List<InitiativeMetricTypeEntity>>())
        } answers {
            firstArg<List<InitiativeMetricTypeEntity>>()
        }

        every {
            metricsDirectoryRepository.findApplicableMetrics(
                autonomousSelected = false,
                copilotSelected = true,
                appealsSelected = false,
            )
        } returns emptyList()

        val request = UpdateInitiativeAgentTypesRequest(agentTypes = setOf("copilot"))

        // When
        val result = updater.updateInitiativeAgentType(initiativeId = initiativeId, request = request)

        // Then
        assertTrue(result.isEmpty())

        verify(exactly = 0) {
            initiativeMetricValueRepository.findValuesForInitiativeMetricsInPeriodRange(
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

    private fun createInitiative(id: Long): AIAgentEntity {
        return AIAgentEntity().apply {
            this.id = id
        }
    }

    private fun createMetric(
        id: UUID,
        autonomousApplicability: Boolean = false,
        copilotApplicability: Boolean = false,
        requiresAppealsWork: Boolean = false,
    ): MetricsDirectoryEntity {
        return MetricsDirectoryEntity(
            name = "Метрика",
            unit = "шт",
            direction = "growth",
            description = "Описание",
            frequency = "monthly",
            autonomousApplicability = autonomousApplicability,
            copilotApplicability = copilotApplicability,
            requiresAppealsWork = requiresAppealsWork,
            active = true,
            updatedBy = 1L,
            updatedAt = LocalDateTime.now(),
        ).apply {
            this.id = id
        }
    }
}

```
