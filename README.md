```java
@ExtendWith(MockKExtension::class)
class InitiativeAgentTypesCreatorTest {

    @MockK
    private lateinit var messageProvider: MessageProvider

    @MockK
    private lateinit var aiAgentRepository: AIAgentRepository

    @MockK
    private lateinit var initiativeMetricTypeRepository: InitiativeMetricTypeRepository

    @MockK
    private lateinit var metricsDirectoryRepository: MetricsDirectoryRepository

    @MockK
    private lateinit var metricResponseBuilder: InitiativeMetricResponseBuilder

    private lateinit var creator: InitiativeAgentTypesCreator

    @BeforeEach
    fun setUp() {
        creator = InitiativeAgentTypesCreator(
            messageProvider = messageProvider,
            aiAgentRepository = aiAgentRepository,
            initiativeMetricTypeRepository = initiativeMetricTypeRepository,
            metricsDirectoryRepository = metricsDirectoryRepository,
            metricResponseBuilder = metricResponseBuilder,
        )
    }

    @Test
    fun `createInitiativeAgentTypes should throw bad request when initiative not found`() {
        // Given
        val initiativeId = 1L

        every {
            aiAgentRepository.findById(initiativeId)
        } returns Optional.empty()

        every {
            messageProvider[INITIATIVE_NOT_FOUND]
        } returns "Initiative {0} not found"

        val request =
            UpdateInitiativeAgentTypesRequest(
                agentTypes = setOf("copilot")
            )

        // When & Then
        assertThrows<AiBadRequestException> {
            creator.createInitiativeAgentTypes(
                initiativeId = initiativeId,
                request = request,
            )
        }

        verify(exactly = 0) {
            initiativeMetricTypeRepository.findAllByAiAgentId(any())
        }

        verify(exactly = 0) {
            metricsDirectoryRepository.findActiveApplicableMetrics(
                autonomousSelected = any(),
                copilotSelected = any(),
                appealsSelected = any(),
            )
        }

        verify(exactly = 0) {
            metricResponseBuilder.buildWithoutValues(any(), any())
        }
    }

    @Test
    fun `createInitiativeAgentTypes should throw bad request when agent type is wrong`() {
        // Given
        val initiativeId = 1L
        val initiative = createInitiative(id = initiativeId)

        every {
            aiAgentRepository.findById(initiativeId)
        } returns Optional.of(initiative)

        every {
            messageProvider[WRONG_INITIATIVE_METRIC_AGENT_TYPE]
        } returns "Wrong agent type {0}"

        val request =
            UpdateInitiativeAgentTypesRequest(
                agentTypes = setOf("wrong")
            )

        // When & Then
        assertThrows<AiBadRequestException> {
            creator.createInitiativeAgentTypes(
                initiativeId = initiativeId,
                request = request,
            )
        }

        verify(exactly = 0) {
            initiativeMetricTypeRepository.findAllByAiAgentId(any())
        }

        verify(exactly = 0) {
            metricsDirectoryRepository.findActiveApplicableMetrics(
                autonomousSelected = any(),
                copilotSelected = any(),
                appealsSelected = any(),
            )
        }

        verify(exactly = 0) {
            metricResponseBuilder.buildWithoutValues(any(), any())
        }
    }

    @Test
    fun `createInitiativeAgentTypes should create agent types and return metrics without values`() {
        // Given
        val initiativeId = 1L
        val initiative = createInitiative(id = initiativeId)

        val metric =
            createMetric(
                id = UUID.randomUUID(),
                autonomousApplicability = true,
                copilotApplicability = true,
            )

        val expectedResponse =
            listOf(
                InitiativeMetricResponse(
                    id = metric.id,
                    name = metric.name,
                    unit = metric.unit,
                    direction = metric.direction,
                    agentTypes = setOf("autonomous", "copilot"),
                    isActive = true,
                    description = metric.description,
                    frequency = metric.frequency,
                    metricValue = null,
                    targetValue = null,
                    periods = emptyList(),
                )
            )

        every {
            aiAgentRepository.findById(initiativeId)
        } returns Optional.of(initiative)

        every {
            initiativeMetricTypeRepository.findAllByAiAgentId(
                initiativeId = initiativeId
            )
        } returns emptyList()

        every {
            initiativeMetricTypeRepository.saveAll(
                any<List<InitiativeMetricTypeEntity>>()
            )
        } answers {
            firstArg<List<InitiativeMetricTypeEntity>>()
        }

        every {
            metricsDirectoryRepository.findActiveApplicableMetrics(
                autonomousSelected = true,
                copilotSelected = true,
                appealsSelected = false,
            )
        } returns listOf(metric)

        every {
            metricResponseBuilder.buildWithoutValues(
                metrics = listOf(metric),
                requestedAgentTypes = setOf(
                    InitiativeMetricAgentType.AUTONOMOUS,
                    InitiativeMetricAgentType.COPILOT,
                ),
            )
        } returns expectedResponse

        val request =
            UpdateInitiativeAgentTypesRequest(
                agentTypes = setOf("autonomous", "copilot")
            )

        // When
        val actualResponse =
            creator.createInitiativeAgentTypes(
                initiativeId = initiativeId,
                request = request,
            )

        // Then
        assertEquals(expectedResponse, actualResponse)

        verify(exactly = 1) {
            initiativeMetricTypeRepository.saveAll(
                match<List<InitiativeMetricTypeEntity>> { entities ->
                    entities.size == 2 &&
                        entities.any { it.aiAgent == initiative && it.agentType == "autonomous" } &&
                        entities.any { it.aiAgent == initiative && it.agentType == "copilot" }
                }
            )
        }

        verify(exactly = 0) {
            initiativeMetricTypeRepository.deleteAll(any<Iterable<InitiativeMetricTypeEntity>>())
        }

        verify(exactly = 1) {
            metricsDirectoryRepository.findActiveApplicableMetrics(
                autonomousSelected = true,
                copilotSelected = true,
                appealsSelected = false,
            )
        }

        verify(exactly = 1) {
            metricResponseBuilder.buildWithoutValues(
                metrics = listOf(metric),
                requestedAgentTypes = setOf(
                    InitiativeMetricAgentType.AUTONOMOUS,
                    InitiativeMetricAgentType.COPILOT,
                ),
            )
        }
    }

    @Test
    fun `createInitiativeAgentTypes should delete removed agent types and create missing agent types`() {
        // Given
        val initiativeId = 1L
        val initiative = createInitiative(id = initiativeId)

        val existingAutonomous =
            InitiativeMetricTypeEntity(
                aiAgent = initiative,
                agentType = "autonomous",
            ).apply {
                id = 10L
            }

        val existingAppeals =
            InitiativeMetricTypeEntity(
                aiAgent = initiative,
                agentType = "appeals",
            ).apply {
                id = 20L
            }

        val metric =
            createMetric(
                id = UUID.randomUUID(),
                autonomousApplicability = true,
                copilotApplicability = true,
            )

        every {
            aiAgentRepository.findById(initiativeId)
        } returns Optional.of(initiative)

        every {
            initiativeMetricTypeRepository.findAllByAiAgentId(
                initiativeId = initiativeId
            )
        } returns listOf(existingAutonomous, existingAppeals)

        every {
            initiativeMetricTypeRepository.deleteAll(
                match<Iterable<InitiativeMetricTypeEntity>> { entities ->
                    entities.toList() == listOf(existingAppeals)
                }
            )
        } just Runs

        every {
            initiativeMetricTypeRepository.saveAll(
                any<List<InitiativeMetricTypeEntity>>()
            )
        } answers {
            firstArg<List<InitiativeMetricTypeEntity>>()
        }

        every {
            metricsDirectoryRepository.findActiveApplicableMetrics(
                autonomousSelected = true,
                copilotSelected = true,
                appealsSelected = false,
            )
        } returns listOf(metric)

        every {
            metricResponseBuilder.buildWithoutValues(
                metrics = listOf(metric),
                requestedAgentTypes = setOf(
                    InitiativeMetricAgentType.AUTONOMOUS,
                    InitiativeMetricAgentType.COPILOT,
                ),
            )
        } returns emptyList()

        val request =
            UpdateInitiativeAgentTypesRequest(
                agentTypes = setOf("autonomous", "copilot")
            )

        // When
        val actualResponse =
            creator.createInitiativeAgentTypes(
                initiativeId = initiativeId,
                request = request,
            )

        // Then
        assertTrue(actualResponse.isEmpty())

        verify(exactly = 1) {
            initiativeMetricTypeRepository.deleteAll(
                match<Iterable<InitiativeMetricTypeEntity>> { entities ->
                    entities.toList() == listOf(existingAppeals)
                }
            )
        }

        verify(exactly = 1) {
            initiativeMetricTypeRepository.saveAll(
                match<List<InitiativeMetricTypeEntity>> { entities ->
                    entities.size == 1 &&
                        entities.first().aiAgent == initiative &&
                        entities.first().agentType == "copilot"
                }
            )
        }

        verify(exactly = 1) {
            metricResponseBuilder.buildWithoutValues(
                metrics = listOf(metric),
                requestedAgentTypes = setOf(
                    InitiativeMetricAgentType.AUTONOMOUS,
                    InitiativeMetricAgentType.COPILOT,
                ),
            )
        }
    }

    @Test
    fun `createInitiativeAgentTypes should not create or delete agent types when already synchronized`() {
        // Given
        val initiativeId = 1L
        val initiative = createInitiative(id = initiativeId)

        val existingAutonomous =
            InitiativeMetricTypeEntity(
                aiAgent = initiative,
                agentType = "autonomous",
            ).apply {
                id = 10L
            }

        val existingCopilot =
            InitiativeMetricTypeEntity(
                aiAgent = initiative,
                agentType = "copilot",
            ).apply {
                id = 20L
            }

        val metric =
            createMetric(
                id = UUID.randomUUID(),
                autonomousApplicability = true,
                copilotApplicability = true,
            )

        every {
            aiAgentRepository.findById(initiativeId)
        } returns Optional.of(initiative)

        every {
            initiativeMetricTypeRepository.findAllByAiAgentId(
                initiativeId = initiativeId
            )
        } returns listOf(existingAutonomous, existingCopilot)

        every {
            metricsDirectoryRepository.findActiveApplicableMetrics(
                autonomousSelected = true,
                copilotSelected = true,
                appealsSelected = false,
            )
        } returns listOf(metric)

        every {
            metricResponseBuilder.buildWithoutValues(
                metrics = listOf(metric),
                requestedAgentTypes = setOf(
                    InitiativeMetricAgentType.AUTONOMOUS,
                    InitiativeMetricAgentType.COPILOT,
                ),
            )
        } returns emptyList()

        val request =
            UpdateInitiativeAgentTypesRequest(
                agentTypes = setOf("autonomous", "copilot")
            )

        // When
        val actualResponse =
            creator.createInitiativeAgentTypes(
                initiativeId = initiativeId,
                request = request,
            )

        // Then
        assertTrue(actualResponse.isEmpty())

        verify(exactly = 0) {
            initiativeMetricTypeRepository.deleteAll(any<Iterable<InitiativeMetricTypeEntity>>())
        }

        verify(exactly = 0) {
            initiativeMetricTypeRepository.saveAll(any<List<InitiativeMetricTypeEntity>>())
        }

        verify(exactly = 1) {
            metricsDirectoryRepository.findActiveApplicableMetrics(
                autonomousSelected = true,
                copilotSelected = true,
                appealsSelected = false,
            )
        }

        verify(exactly = 1) {
            metricResponseBuilder.buildWithoutValues(
                metrics = listOf(metric),
                requestedAgentTypes = setOf(
                    InitiativeMetricAgentType.AUTONOMOUS,
                    InitiativeMetricAgentType.COPILOT,
                ),
            )
        }
    }

    @Test
    fun `createInitiativeAgentTypes should pass correct flags for appeals only`() {
        // Given
        val initiativeId = 1L
        val initiative = createInitiative(id = initiativeId)

        val metric =
            createMetric(
                id = UUID.randomUUID(),
                requiresAppealsWork = true,
            )

        every {
            aiAgentRepository.findById(initiativeId)
        } returns Optional.of(initiative)

        every {
            initiativeMetricTypeRepository.findAllByAiAgentId(
                initiativeId = initiativeId
            )
        } returns emptyList()

        every {
            initiativeMetricTypeRepository.saveAll(
                any<List<InitiativeMetricTypeEntity>>()
            )
        } answers {
            firstArg<List<InitiativeMetricTypeEntity>>()
        }

        every {
            metricsDirectoryRepository.findActiveApplicableMetrics(
                autonomousSelected = false,
                copilotSelected = false,
                appealsSelected = true,
            )
        } returns listOf(metric)

        every {
            metricResponseBuilder.buildWithoutValues(
                metrics = listOf(metric),
                requestedAgentTypes = setOf(InitiativeMetricAgentType.APPEALS),
            )
        } returns emptyList()

        val request =
            UpdateInitiativeAgentTypesRequest(
                agentTypes = setOf("appeals")
            )

        // When
        creator.createInitiativeAgentTypes(
            initiativeId = initiativeId,
            request = request,
        )

        // Then
        verify(exactly = 1) {
            metricsDirectoryRepository.findActiveApplicableMetrics(
                autonomousSelected = false,
                copilotSelected = false,
                appealsSelected = true,
            )
        }

        verify(exactly = 1) {
            initiativeMetricTypeRepository.saveAll(
                match<List<InitiativeMetricTypeEntity>> { entities ->
                    entities.size == 1 &&
                        entities.first().aiAgent == initiative &&
                        entities.first().agentType == "appeals"
                }
            )
        }
    }

    private fun createInitiative(
        id: Long,
    ): AIAgentEntity {
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
