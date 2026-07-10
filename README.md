```java
@ExtendWith(MockKExtension::class)
class InitiativeMetricValueSaverTest {

    @MockK
    private lateinit var messageProvider: MessageProvider

    @MockK
    private lateinit var initiativeMetricTypeRepository: InitiativeMetricTypeRepository

    @MockK
    private lateinit var initiativeMetricValueRepository: InitiativeMetricValueRepository

    @MockK
    private lateinit var metricsDirectoryRepository: MetricsDirectoryRepository

    @InjectMockKs
    private lateinit var service: InitiativeMetricValueCreator

    @Test
    fun `saveInitiativeMetricValue should create metric value when value does not exist`() {
        // Given
        val initiativeId = 1L
        val initiativeMetricTypeId = 10L
        val metricId = UUID.randomUUID()
        val periodMonth = YearMonth.now().atDay(1)

        val request = SaveInitiativeMetricValuesRequest(
            metricsValues = listOf(
                SaveInitiativeMetricValueRequest(
                    agentType = "copilot",
                    metricId = metricId,
                    metricValue = BigDecimal("10"),
                    targetValue = BigDecimal("20"),
                )
            )
        )

        val metricDirectory = MetricsDirectoryEntity().also {
            it.id = metricId
        }

        val initiativeMetricType = InitiativeMetricTypeEntity(
            aiAgent = AIAgentEntity().also {
                it.id = initiativeId
            },
            agentType = "copilot",
        ).also {
            it.id = initiativeMetricTypeId
        }

        val savedMetricValues = mutableListOf<InitiativeMetricValueEntity>()

        every {
            initiativeMetricTypeRepository.existsByAiAgentId(initiativeId)
        } returns true

        every {
            metricsDirectoryRepository.findAllById(setOf(metricId))
        } returns listOf(metricDirectory)

        every {
            initiativeMetricTypeRepository.findAllByAiAgentIdAndAgentTypeIn(
                initiativeId = initiativeId,
                agentTypes = setOf("copilot"),
            )
        } returns listOf(initiativeMetricType)

        every {
            initiativeMetricValueRepository.findAllByInitiativeMetricTypeIdsAndMetricDirectoryIdsAndPeriodMonth(
                initiativeMetricTypeIds = setOf(initiativeMetricTypeId),
                metricDirectoryIds = setOf(metricId),
                periodMonth = periodMonth,
            )
        } returns emptyList()

        every {
            initiativeMetricValueRepository.saveAll(any<Iterable<InitiativeMetricValueEntity>>())
        } answers {
            firstArg<Iterable<InitiativeMetricValueEntity>>()
                .toList()
                .also {
                    savedMetricValues.clear()
                    savedMetricValues.addAll(it)
                }
        }

        // When
        val response = service.saveInitiativeMetricValue(
            initiativeId = initiativeId,
            request = request,
        )

        // Then
        assertEquals(0, response.code)
        assertEquals("Metrics saved", response.message)

        val savedMetricValue = savedMetricValues.single()

        Assertions.assertSame(
            initiativeMetricType,
            savedMetricValue.initiativeMetricType,
        )
        Assertions.assertSame(
            metricDirectory,
            savedMetricValue.metricDirectory,
        )
        assertEquals(periodMonth, savedMetricValue.periodMonth)
        assertEquals(BigDecimal("10"), savedMetricValue.metricValue)
        assertEquals(BigDecimal("20"), savedMetricValue.targetValue)

        verify(exactly = 1) {
            metricsDirectoryRepository.findAllById(setOf(metricId))
        }

        verify(exactly = 1) {
            initiativeMetricTypeRepository.findAllByAiAgentIdAndAgentTypeIn(
                initiativeId = initiativeId,
                agentTypes = setOf("copilot"),
            )
        }

        verify(exactly = 1) {
            initiativeMetricValueRepository.findAllByInitiativeMetricTypeIdsAndMetricDirectoryIdsAndPeriodMonth(
                initiativeMetricTypeIds = setOf(initiativeMetricTypeId),
                metricDirectoryIds = setOf(metricId),
                periodMonth = periodMonth,
            )
        }

        verify(exactly = 1) {
            initiativeMetricValueRepository.saveAll(any<Iterable<InitiativeMetricValueEntity>>())
        }
    }

    @Test
    fun `saveInitiativeMetricValue should update metric value when value already exists`() {
        // Given
        val initiativeId = 1L
        val initiativeMetricTypeId = 10L
        val metricId = UUID.randomUUID()
        val periodMonth = YearMonth.now().atDay(1)

        val request = SaveInitiativeMetricValuesRequest(
            metricsValues = listOf(
                SaveInitiativeMetricValueRequest(
                    agentType = "copilot",
                    metricId = metricId,
                    metricValue = BigDecimal("15"),
                    targetValue = BigDecimal("25"),
                )
            )
        )

        val metricDirectory = MetricsDirectoryEntity().also {
            it.id = metricId
        }

        val initiativeMetricType = InitiativeMetricTypeEntity(
            aiAgent = AIAgentEntity().also {
                it.id = initiativeId
            },
            agentType = "copilot",
        ).also {
            it.id = initiativeMetricTypeId
        }

        val existingMetricValue = InitiativeMetricValueEntity(
            initiativeMetricType = initiativeMetricType,
            metricDirectory = metricDirectory,
            metricValue = BigDecimal("10"),
            targetValue = BigDecimal("20"),
        ).also {
            it.id = 100L
            it.periodMonth = periodMonth
        }

        val savedMetricValues = mutableListOf<InitiativeMetricValueEntity>()

        every {
            initiativeMetricTypeRepository.existsByAiAgentId(initiativeId)
        } returns true

        every {
            metricsDirectoryRepository.findAllById(setOf(metricId))
        } returns listOf(metricDirectory)

        every {
            initiativeMetricTypeRepository.findAllByAiAgentIdAndAgentTypeIn(
                initiativeId = initiativeId,
                agentTypes = setOf("copilot"),
            )
        } returns listOf(initiativeMetricType)

        every {
            initiativeMetricValueRepository.findAllByInitiativeMetricTypeIdsAndMetricDirectoryIdsAndPeriodMonth(
                initiativeMetricTypeIds = setOf(initiativeMetricTypeId),
                metricDirectoryIds = setOf(metricId),
                periodMonth = periodMonth,
            )
        } returns listOf(existingMetricValue)

        every {
            initiativeMetricValueRepository.saveAll(any<Iterable<InitiativeMetricValueEntity>>())
        } answers {
            firstArg<Iterable<InitiativeMetricValueEntity>>()
                .toList()
                .also {
                    savedMetricValues.clear()
                    savedMetricValues.addAll(it)
                }
        }

        // When
        val response = service.saveInitiativeMetricValue(
            initiativeId = initiativeId,
            request = request,
        )

        // Then
        assertEquals(0, response.code)
        assertEquals("Metrics saved", response.message)

        val savedMetricValue = savedMetricValues.single()

        Assertions.assertSame(existingMetricValue, savedMetricValue)
        assertEquals(periodMonth, existingMetricValue.periodMonth)
        assertEquals(BigDecimal("15"), existingMetricValue.metricValue)
        assertEquals(BigDecimal("25"), existingMetricValue.targetValue)

        verify(exactly = 1) {
            initiativeMetricValueRepository.saveAll(any<Iterable<InitiativeMetricValueEntity>>())
        }
    }

    @Test
    fun `saveInitiativeMetricValue should throw bad request when agent type is wrong`() {
        // Given
        val initiativeId = 1L

        val request = SaveInitiativeMetricValuesRequest(
            metricsValues = listOf(
                SaveInitiativeMetricValueRequest(
                    agentType = "wrong",
                    metricId = UUID.randomUUID(),
                    metricValue = BigDecimal("10"),
                    targetValue = BigDecimal("20"),
                )
            )
        )

        every {
            initiativeMetricTypeRepository.existsByAiAgentId(initiativeId)
        } returns true

        every {
            messageProvider[Metadata.ErrorMessages.WRONG_INITIATIVE_METRIC_AGENT_TYPE]
        } returns "Недопустимый режим работы инициативы: {0}"

        // When & Then
        assertThrows<AiBadRequestException> {
            service.saveInitiativeMetricValue(
                initiativeId = initiativeId,
                request = request,
            )
        }

        verify(exactly = 0) {
            metricsDirectoryRepository.findAllById(any<Iterable<UUID>>())
        }

        verify(exactly = 0) {
            initiativeMetricValueRepository.saveAll(any<Iterable<InitiativeMetricValueEntity>>())
        }
    }

    @Test
    fun `saveInitiativeMetricValue should throw bad request when metric not found`() {
        // Given
        val initiativeId = 1L
        val initiativeMetricTypeId = 10L
        val metricId = UUID.randomUUID()
        val periodMonth = YearMonth.now().atDay(1)

        val request = SaveInitiativeMetricValuesRequest(
            metricsValues = listOf(
                SaveInitiativeMetricValueRequest(
                    agentType = "copilot",
                    metricId = metricId,
                    metricValue = BigDecimal("10"),
                    targetValue = BigDecimal("20"),
                )
            )
        )

        val initiativeMetricType = InitiativeMetricTypeEntity(
            aiAgent = AIAgentEntity().also {
                it.id = initiativeId
            },
            agentType = "copilot",
        ).also {
            it.id = initiativeMetricTypeId
        }

        every {
            initiativeMetricTypeRepository.existsByAiAgentId(initiativeId)
        } returns true

        every {
            metricsDirectoryRepository.findAllById(setOf(metricId))
        } returns emptyList()

        every {
            initiativeMetricTypeRepository.findAllByAiAgentIdAndAgentTypeIn(
                initiativeId = initiativeId,
                agentTypes = setOf("copilot"),
            )
        } returns listOf(initiativeMetricType)

        every {
            initiativeMetricValueRepository.findAllByInitiativeMetricTypeIdsAndMetricDirectoryIdsAndPeriodMonth(
                initiativeMetricTypeIds = setOf(initiativeMetricTypeId),
                metricDirectoryIds = setOf(metricId),
                periodMonth = periodMonth,
            )
        } returns emptyList()

        every {
            messageProvider[Metadata.ErrorMessages.INITIATIVE_METRIC_NOT_FOUND]
        } returns "Метрика с идентификатором {0} не найдена"

        // When & Then
        assertThrows<AiBadRequestException> {
            service.saveInitiativeMetricValue(
                initiativeId = initiativeId,
                request = request,
            )
        }

        verify(exactly = 0) {
            initiativeMetricValueRepository.saveAll(any<Iterable<InitiativeMetricValueEntity>>())
        }
    }

    @Test
    fun `saveInitiativeMetricValue should throw conflict when initiative has no metric types`() {
        // Given
        val initiativeId = 1L
        val metricId = UUID.randomUUID()

        val request = SaveInitiativeMetricValuesRequest(
            metricsValues = listOf(
                SaveInitiativeMetricValueRequest(
                    agentType = "copilot",
                    metricId = metricId,
                    metricValue = BigDecimal("10"),
                    targetValue = BigDecimal("20"),
                )
            )
        )

        every {
            initiativeMetricTypeRepository.existsByAiAgentId(initiativeId)
        } returns false

        every {
            messageProvider[Metadata.ErrorMessages.INITIATIVE_METRIC_TYPES_NOT_FOUND]
        } returns "Для инициативы с идентификатором {0} не найдены режимы работы"

        // When & Then
        assertThrows<AiConflictException> {
            service.saveInitiativeMetricValue(
                initiativeId = initiativeId,
                request = request,
            )
        }

        verify(exactly = 0) {
            metricsDirectoryRepository.findAllById(any<Iterable<UUID>>())
        }

        verify(exactly = 0) {
            initiativeMetricTypeRepository.findAllByAiAgentIdAndAgentTypeIn(
                initiativeId = any(),
                agentTypes = any(),
            )
        }

        verify(exactly = 0) {
            initiativeMetricValueRepository.saveAll(any<Iterable<InitiativeMetricValueEntity>>())
        }
    }

    @Test
    fun `saveInitiativeMetricValue should throw conflict when initiative agent type not found`() {
        // Given
        val initiativeId = 1L
        val metricId = UUID.randomUUID()

        val request = SaveInitiativeMetricValuesRequest(
            metricsValues = listOf(
                SaveInitiativeMetricValueRequest(
                    agentType = "copilot",
                    metricId = metricId,
                    metricValue = BigDecimal("10"),
                    targetValue = BigDecimal("20"),
                )
            )
        )

        val metricDirectory = MetricsDirectoryEntity().also {
            it.id = metricId
        }

        every {
            initiativeMetricTypeRepository.existsByAiAgentId(initiativeId)
        } returns true

        every {
            metricsDirectoryRepository.findAllById(setOf(metricId))
        } returns listOf(metricDirectory)

        every {
            initiativeMetricTypeRepository.findAllByAiAgentIdAndAgentTypeIn(
                initiativeId = initiativeId,
                agentTypes = setOf("copilot"),
            )
        } returns emptyList()

        every {
            messageProvider[Metadata.ErrorMessages.INITIATIVE_METRIC_AGENT_TYPE_NOT_FOUND]
        } returns "Для инициативы с идентификатором {0} не найден режим работы: {1}"

        // When & Then
        assertThrows<AiConflictException> {
            service.saveInitiativeMetricValue(
                initiativeId = initiativeId,
                request = request,
            )
        }

        verify(exactly = 0) {
            initiativeMetricValueRepository.saveAll(any<Iterable<InitiativeMetricValueEntity>>())
        }
    }
}
```
