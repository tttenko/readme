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
            metricsDirectoryRepository.findById(metricId)
        } returns Optional.of(metricDirectory)

        every {
            initiativeMetricTypeRepository.findByAiAgentIdAndAgentType(
                initiativeId = initiativeId,
                agentType = "copilot",
            )
        } returns initiativeMetricType

        every {
            initiativeMetricValueRepository.findByInitiativeMetricTypeIdAndMetricDirectoryIdAndPeriodMonth(
                initiativeMetricTypeId = initiativeMetricTypeId,
                metricDirectoryId = metricId,
                periodMonth = any(),
            )
        } returns null

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
        assertEquals(
            YearMonth.now().atDay(1),
            savedMetricValue.periodMonth,
        )
        assertEquals(
            BigDecimal("10"),
            savedMetricValue.metricValue,
        )
        assertEquals(
            BigDecimal("20"),
            savedMetricValue.targetValue,
        )

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
        }

        val savedMetricValues = mutableListOf<InitiativeMetricValueEntity>()

        every {
            initiativeMetricTypeRepository.existsByAiAgentId(initiativeId)
        } returns true

        every {
            metricsDirectoryRepository.findById(metricId)
        } returns Optional.of(metricDirectory)

        every {
            initiativeMetricTypeRepository.findByAiAgentIdAndAgentType(
                initiativeId = initiativeId,
                agentType = "copilot",
            )
        } returns initiativeMetricType

        every {
            initiativeMetricValueRepository.findByInitiativeMetricTypeIdAndMetricDirectoryIdAndPeriodMonth(
                initiativeMetricTypeId = initiativeMetricTypeId,
                metricDirectoryId = metricId,
                periodMonth = any(),
            )
        } returns existingMetricValue

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
            existingMetricValue,
            savedMetricValue,
        )
        assertEquals(
            YearMonth.now().atDay(1),
            existingMetricValue.periodMonth,
        )
        assertEquals(
            BigDecimal("15"),
            existingMetricValue.metricValue,
        )
        assertEquals(
            BigDecimal("25"),
            existingMetricValue.targetValue,
        )

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
            initiativeMetricValueRepository.saveAll(any<Iterable<InitiativeMetricValueEntity>>())
        }
    }

    @Test
    fun `saveInitiativeMetricValue should throw bad request when metric not found`() {
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
        } returns true

        every {
            metricsDirectoryRepository.findById(metricId)
        } returns Optional.empty()

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
            metricsDirectoryRepository.findById(any())
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
            metricsDirectoryRepository.findById(metricId)
        } returns Optional.of(metricDirectory)

        every {
            initiativeMetricTypeRepository.findByAiAgentIdAndAgentType(
                initiativeId = initiativeId,
                agentType = "copilot",
            )
        } returns null

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
