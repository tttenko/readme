```java
@ExtendWith(MockKExtension::class)
class InitiativeMetricValueCreatorTest {

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

    @InjectMockKs
    private lateinit var service: InitiativeMetricValueCreator

    @Test
    fun `saveInitiativeMetricValue should create metric value when value does not exist`() {
        // Given
        val initiativeId = 1L
        val initiativeMetricTypeId = 10L
        val metricId = UUID.randomUUID()
        val periodMonth = currentPeriodMonth()

        val request = createRequest(
            metricId = metricId,
            agentType = " copilot ",
            metricValue = "10",
            targetValue = "20",
        )

        val metricDirectory = createMetricDirectory(metricId)

        val initiativeMetricType = createInitiativeMetricType(
            initiativeId = initiativeId,
            initiativeMetricTypeId = initiativeMetricTypeId,
            agentType = "copilot",
        )

        stubSuccessfulReadPath(
            initiativeId = initiativeId,
            initiativeMetricTypeId = initiativeMetricTypeId,
            metricId = metricId,
            periodMonth = periodMonth,
            metricDirectory = metricDirectory,
            initiativeMetricType = initiativeMetricType,
        )

        val savedValues = stubSaveAllAndFlush()

        // When
        val response = service.saveInitiativeMetricValue(
            initiativeId = initiativeId,
            request = request,
        )

        // Then
        assertEquals(0, response.code)
        assertEquals("Metrics saved", response.message)

        val savedValue = savedValues.single()

        assertSame(
            initiativeMetricType,
            savedValue.initiativeMetricType,
        )
        assertSame(
            metricDirectory,
            savedValue.metricDirectory,
        )

        assertEquals(periodMonth, savedValue.periodMonth)
        assertEquals(BigDecimal("10"), savedValue.metricValue)
        assertEquals(BigDecimal("20"), savedValue.targetValue)

        verify(exactly = 1) {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdAndAgentTypeIn(
                    initiativeId = initiativeId,
                    agentTypes = setOf("copilot"),
                )
        }

        verify(exactly = 1) {
            initiativeMetricValueRepository
                .saveAllAndFlush<InitiativeMetricValueEntity>(any())
        }
    }

    @Test
    fun `saveInitiativeMetricValue should update metric value when value already exists`() {
        // Given
        val initiativeId = 1L
        val initiativeMetricTypeId = 10L
        val metricId = UUID.randomUUID()
        val periodMonth = currentPeriodMonth()

        val request = createRequest(
            metricId = metricId,
            metricValue = "15",
            targetValue = "25",
        )

        val metricDirectory = createMetricDirectory(metricId)

        val initiativeMetricType = createInitiativeMetricType(
            initiativeId = initiativeId,
            initiativeMetricTypeId = initiativeMetricTypeId,
            agentType = "copilot",
        )

        val existingMetricValue = InitiativeMetricValueEntity(
            initiativeMetricType = initiativeMetricType,
            metricDirectory = metricDirectory,
            metricValue = BigDecimal("10"),
            targetValue = BigDecimal("20"),
        ).also {
            it.id = 100L
            it.periodMonth = periodMonth
        }

        stubSuccessfulReadPath(
            initiativeId = initiativeId,
            initiativeMetricTypeId = initiativeMetricTypeId,
            metricId = metricId,
            periodMonth = periodMonth,
            metricDirectory = metricDirectory,
            initiativeMetricType = initiativeMetricType,
            existingValues = listOf(existingMetricValue),
        )

        val savedValues = stubSaveAllAndFlush()

        // When
        val response = service.saveInitiativeMetricValue(
            initiativeId = initiativeId,
            request = request,
        )

        // Then
        assertEquals(0, response.code)
        assertEquals("Metrics saved", response.message)

        val savedValue = savedValues.single()

        assertSame(existingMetricValue, savedValue)
        assertEquals(100L, savedValue.id)
        assertEquals(periodMonth, savedValue.periodMonth)
        assertEquals(BigDecimal("15"), savedValue.metricValue)
        assertEquals(BigDecimal("25"), savedValue.targetValue)
    }

    @Test
    fun `saveInitiativeMetricValue should throw bad request when request contains duplicate`() {
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
                ),
                SaveInitiativeMetricValueRequest(
                    agentType = " copilot ",
                    metricId = metricId,
                    metricValue = BigDecimal("15"),
                    targetValue = BigDecimal("25"),
                ),
            )
        )

        every {
            messageProvider[
                Metadata.ErrorMessages
                    .INITIATIVE_METRIC_VALUE_ALREADY_EXISTS
            ]
        } returns "Метрика {0} для режима {1} дублируется"

        // When
        val exception = assertThrows<AiBadRequestException> {
            service.saveInitiativeMetricValue(
                initiativeId = initiativeId,
                request = request,
            )
        }

        // Then
        assertEquals(
            "Метрика $metricId для режима copilot дублируется",
            exception.message,
        )

        verify(exactly = 0) {
            initiativeMetricTypeRepository.existsByAiAgentId(any())
        }

        verify(exactly = 0) {
            metricsDirectoryRepository.findAllById(any<Iterable<UUID>>())
        }

        verify(exactly = 0) {
            initiativeMetricValueRepository
                .saveAllAndFlush<InitiativeMetricValueEntity>(any())
        }
    }

    @Test
    fun `saveInitiativeMetricValue should throw conflict when initiative has no metric types`() {
        // Given
        val initiativeId = 1L
        val metricId = UUID.randomUUID()

        val request = createRequest(metricId)

        every {
            initiativeMetricTypeRepository.existsByAiAgentId(initiativeId)
        } returns false

        every {
            messageProvider[
                Metadata.ErrorMessages
                    .INITIATIVE_METRIC_TYPES_NOT_FOUND
            ]
        } returns "Для инициативы {0} не найдены режимы работы"

        // When
        val exception = assertThrows<AiConflictException> {
            service.saveInitiativeMetricValue(
                initiativeId = initiativeId,
                request = request,
            )
        }

        // Then
        assertEquals(
            "Для инициативы $initiativeId не найдены режимы работы",
            exception.message,
        )

        verify(exactly = 0) {
            metricsDirectoryRepository.findAllById(any<Iterable<UUID>>())
        }

        verify(exactly = 0) {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdAndAgentTypeIn(
                    initiativeId = any(),
                    agentTypes = any(),
                )
        }

        verify(exactly = 0) {
            initiativeMetricValueRepository
                .saveAllAndFlush<InitiativeMetricValueEntity>(any())
        }
    }

    @Test
    fun `saveInitiativeMetricValue should throw bad request when agent type is wrong`() {
        // Given
        val initiativeId = 1L
        val metricId = UUID.randomUUID()

        val request = createRequest(
            metricId = metricId,
            agentType = " wrong ",
        )

        every {
            initiativeMetricTypeRepository.existsByAiAgentId(initiativeId)
        } returns true

        every {
            messageProvider[
                Metadata.ErrorMessages
                    .WRONG_INITIATIVE_METRIC_AGENT_TYPE
            ]
        } returns "Недопустимый режим работы инициативы: {0}"

        // When
        val exception = assertThrows<AiBadRequestException> {
            service.saveInitiativeMetricValue(
                initiativeId = initiativeId,
                request = request,
            )
        }

        // Then
        assertEquals(
            "Недопустимый режим работы инициативы: wrong",
            exception.message,
        )

        verify(exactly = 0) {
            metricsDirectoryRepository.findAllById(any<Iterable<UUID>>())
        }

        verify(exactly = 0) {
            initiativeMetricValueRepository
                .saveAllAndFlush<InitiativeMetricValueEntity>(any())
        }
    }

    @Test
    fun `saveInitiativeMetricValue should throw bad request when metric does not exist`() {
        // Given
        val initiativeId = 1L
        val metricId = UUID.randomUUID()

        val request = createRequest(metricId)

        every {
            initiativeMetricTypeRepository.existsByAiAgentId(initiativeId)
        } returns true

        every {
            metricsDirectoryRepository.findAllById(setOf(metricId))
        } returns emptyList()

        every {
            messageProvider[
                Metadata.ErrorMessages.INITIATIVE_METRIC_NOT_FOUND
            ]
        } returns "Метрика {0} не найдена"

        // When
        val exception = assertThrows<AiBadRequestException> {
            service.saveInitiativeMetricValue(
                initiativeId = initiativeId,
                request = request,
            )
        }

        // Then
        assertEquals(
            "Метрика $metricId не найдена",
            exception.message,
        )

        verify(exactly = 0) {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdAndAgentTypeIn(
                    initiativeId = any(),
                    agentTypes = any(),
                )
        }

        verify(exactly = 0) {
            initiativeMetricValueRepository
                .saveAllAndFlush<InitiativeMetricValueEntity>(any())
        }
    }

    @Test
    fun `saveInitiativeMetricValue should throw conflict when requested agent type is not assigned to initiative`() {
        // Given
        val initiativeId = 1L
        val metricId = UUID.randomUUID()

        val request = createRequest(metricId)

        val metricDirectory = createMetricDirectory(metricId)

        every {
            initiativeMetricTypeRepository.existsByAiAgentId(initiativeId)
        } returns true

        every {
            metricsDirectoryRepository.findAllById(setOf(metricId))
        } returns listOf(metricDirectory)

        every {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdAndAgentTypeIn(
                    initiativeId = initiativeId,
                    agentTypes = setOf("copilot"),
                )
        } returns emptyList()

        every {
            messageProvider[
                Metadata.ErrorMessages
                    .INITIATIVE_METRIC_AGENT_TYPE_NOT_FOUND
            ]
        } returns "Для инициативы {0} не найден режим работы: {1}"

        // When
        val exception = assertThrows<AiConflictException> {
            service.saveInitiativeMetricValue(
                initiativeId = initiativeId,
                request = request,
            )
        }

        // Then
        assertEquals(
            "Для инициативы $initiativeId не найден режим работы: copilot",
            exception.message,
        )

        verify(exactly = 0) {
            initiativeMetricValueRepository
                .findAllByInitiativeMetricTypeIdsAndMetricDirectoryIdsAndPeriodMonth(
                    initiativeMetricTypeIds = any(),
                    metricDirectoryIds = any(),
                    periodMonth = any(),
                )
        }

        verify(exactly = 0) {
            initiativeMetricValueRepository
                .saveAllAndFlush<InitiativeMetricValueEntity>(any())
        }
    }

    @Test
    fun `saveInitiativeMetricValue should fail when metric directory id is null`() {
        // Given
        val initiativeId = 1L
        val metricId = UUID.randomUUID()

        val request = createRequest(metricId)

        val metricDirectory = MetricsDirectoryEntity().also {
            it.id = null
        }

        every {
            initiativeMetricTypeRepository.existsByAiAgentId(initiativeId)
        } returns true

        every {
            metricsDirectoryRepository.findAllById(setOf(metricId))
        } returns listOf(metricDirectory)

        // When
        val exception = assertThrows<IllegalArgumentException> {
            service.saveInitiativeMetricValue(
                initiativeId = initiativeId,
                request = request,
            )
        }

        // Then
        assertEquals(
            "metricDirectory.id must not be null",
            exception.message,
        )

        verify(exactly = 0) {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdAndAgentTypeIn(
                    initiativeId = any(),
                    agentTypes = any(),
                )
        }
    }

    @Test
    fun `saveInitiativeMetricValue should fail when initiative metric type agent type is null`() {
        // Given
        val initiativeId = 1L
        val initiativeMetricTypeId = 10L
        val metricId = UUID.randomUUID()

        val request = createRequest(metricId)
        val metricDirectory = createMetricDirectory(metricId)

        val initiativeMetricType = createInitiativeMetricType(
            initiativeId = initiativeId,
            initiativeMetricTypeId = initiativeMetricTypeId,
            agentType = null,
        )

        every {
            initiativeMetricTypeRepository.existsByAiAgentId(initiativeId)
        } returns true

        every {
            metricsDirectoryRepository.findAllById(setOf(metricId))
        } returns listOf(metricDirectory)

        every {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdAndAgentTypeIn(
                    initiativeId = initiativeId,
                    agentTypes = setOf("copilot"),
                )
        } returns listOf(initiativeMetricType)

        // When
        val exception = assertThrows<IllegalArgumentException> {
            service.saveInitiativeMetricValue(
                initiativeId = initiativeId,
                request = request,
            )
        }

        // Then
        assertEquals(
            "initiativeMetricType.agentType must not be null",
            exception.message,
        )

        verify(exactly = 0) {
            initiativeMetricValueRepository
                .findAllByInitiativeMetricTypeIdsAndMetricDirectoryIdsAndPeriodMonth(
                    initiativeMetricTypeIds = any(),
                    metricDirectoryIds = any(),
                    periodMonth = any(),
                )
        }
    }

    @Test
    fun `saveInitiativeMetricValue should fail when initiative metric type id is null`() {
        // Given
        val initiativeId = 1L
        val metricId = UUID.randomUUID()

        val request = createRequest(metricId)
        val metricDirectory = createMetricDirectory(metricId)

        val initiativeMetricType = createInitiativeMetricType(
            initiativeId = initiativeId,
            initiativeMetricTypeId = null,
            agentType = "copilot",
        )

        every {
            initiativeMetricTypeRepository.existsByAiAgentId(initiativeId)
        } returns true

        every {
            metricsDirectoryRepository.findAllById(setOf(metricId))
        } returns listOf(metricDirectory)

        every {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdAndAgentTypeIn(
                    initiativeId = initiativeId,
                    agentTypes = setOf("copilot"),
                )
        } returns listOf(initiativeMetricType)

        // When
        val exception = assertThrows<IllegalArgumentException> {
            service.saveInitiativeMetricValue(
                initiativeId = initiativeId,
                request = request,
            )
        }

        // Then
        assertEquals(
            "initiativeMetricType.id must not be null",
            exception.message,
        )

        verify(exactly = 0) {
            initiativeMetricValueRepository
                .findAllByInitiativeMetricTypeIdsAndMetricDirectoryIdsAndPeriodMonth(
                    initiativeMetricTypeIds = any(),
                    metricDirectoryIds = any(),
                    periodMonth = any(),
                )
        }
    }

    @Test
    fun `saveInitiativeMetricValue should fail when existing value has null initiative metric type`() {
        // Given
        val initiativeId = 1L
        val initiativeMetricTypeId = 10L
        val metricId = UUID.randomUUID()
        val periodMonth = currentPeriodMonth()

        val request = createRequest(metricId)
        val metricDirectory = createMetricDirectory(metricId)

        val initiativeMetricType = createInitiativeMetricType(
            initiativeId = initiativeId,
            initiativeMetricTypeId = initiativeMetricTypeId,
            agentType = "copilot",
        )

        val existingValue = InitiativeMetricValueEntity(
            initiativeMetricType = null,
            metricDirectory = metricDirectory,
        )

        stubSuccessfulReadPath(
            initiativeId = initiativeId,
            initiativeMetricTypeId = initiativeMetricTypeId,
            metricId = metricId,
            periodMonth = periodMonth,
            metricDirectory = metricDirectory,
            initiativeMetricType = initiativeMetricType,
            existingValues = listOf(existingValue),
        )

        // When
        val exception = assertThrows<IllegalArgumentException> {
            service.saveInitiativeMetricValue(
                initiativeId = initiativeId,
                request = request,
            )
        }

        // Then
        assertEquals(
            "initiativeMetricType.id must not be null",
            exception.message,
        )

        verify(exactly = 0) {
            initiativeMetricValueRepository
                .saveAllAndFlush<InitiativeMetricValueEntity>(any())
        }
    }

    @Test
    fun `saveInitiativeMetricValue should fail when existing value has null metric directory`() {
        // Given
        val initiativeId = 1L
        val initiativeMetricTypeId = 10L
        val metricId = UUID.randomUUID()
        val periodMonth = currentPeriodMonth()

        val request = createRequest(metricId)
        val metricDirectory = createMetricDirectory(metricId)

        val initiativeMetricType = createInitiativeMetricType(
            initiativeId = initiativeId,
            initiativeMetricTypeId = initiativeMetricTypeId,
            agentType = "copilot",
        )

        val existingValue = InitiativeMetricValueEntity(
            initiativeMetricType = initiativeMetricType,
            metricDirectory = null,
        )

        stubSuccessfulReadPath(
            initiativeId = initiativeId,
            initiativeMetricTypeId = initiativeMetricTypeId,
            metricId = metricId,
            periodMonth = periodMonth,
            metricDirectory = metricDirectory,
            initiativeMetricType = initiativeMetricType,
            existingValues = listOf(existingValue),
        )

        // When
        val exception = assertThrows<IllegalArgumentException> {
            service.saveInitiativeMetricValue(
                initiativeId = initiativeId,
                request = request,
            )
        }

        // Then
        assertEquals(
            "metricDirectory.id must not be null",
            exception.message,
        )

        verify(exactly = 0) {
            initiativeMetricValueRepository
                .saveAllAndFlush<InitiativeMetricValueEntity>(any())
        }
    }

    @Test
    fun `saveInitiativeMetricValue should convert unique constraint violation to bad request`() {
        // Given
        val initiativeId = 1L
        val initiativeMetricTypeId = 10L
        val metricId = UUID.randomUUID()
        val periodMonth = currentPeriodMonth()

        val request = createRequest(metricId)
        val metricDirectory = createMetricDirectory(metricId)

        val initiativeMetricType = createInitiativeMetricType(
            initiativeId = initiativeId,
            initiativeMetricTypeId = initiativeMetricTypeId,
            agentType = "copilot",
        )

        stubSuccessfulReadPath(
            initiativeId = initiativeId,
            initiativeMetricTypeId = initiativeMetricTypeId,
            metricId = metricId,
            periodMonth = periodMonth,
            metricDirectory = metricDirectory,
            initiativeMetricType = initiativeMetricType,
        )

        val constraintViolationException = ConstraintViolationException(
            "Duplicate metric value",
            SQLException("Duplicate key"),
            Metadata.ErrorMessages
                .INITIATIVE_METRIC_VALUE_ALREADY_EXISTS,
        )

        val dataIntegrityViolationException =
            DataIntegrityViolationException(
                "Could not save metric value",
                constraintViolationException,
            )

        every {
            initiativeMetricValueRepository
                .saveAllAndFlush<InitiativeMetricValueEntity>(any())
        } throws dataIntegrityViolationException

        every {
            messageProvider[
                Metadata.ErrorMessages
                    .INITIATIVE_METRIC_VALUE_DUPLICATE
            ]
        } returns "Дубликат метрики {0} для режима {1}"

        // When
        val exception = assertThrows<AiBadRequestException> {
            service.saveInitiativeMetricValue(
                initiativeId = initiativeId,
                request = request,
            )
        }

        // Then
        assertEquals(
            "Дубликат метрики $metricId для режима copilot",
            exception.message,
        )
    }

    @Test
    fun `saveInitiativeMetricValue should detect unique violation by exception message`() {
        // Given
        val initiativeId = 1L
        val initiativeMetricTypeId = 10L
        val metricId = UUID.randomUUID()
        val periodMonth = currentPeriodMonth()

        val request = createRequest(metricId)
        val metricDirectory = createMetricDirectory(metricId)

        val initiativeMetricType = createInitiativeMetricType(
            initiativeId = initiativeId,
            initiativeMetricTypeId = initiativeMetricTypeId,
            agentType = "copilot",
        )

        stubSuccessfulReadPath(
            initiativeId = initiativeId,
            initiativeMetricTypeId = initiativeMetricTypeId,
            metricId = metricId,
            periodMonth = periodMonth,
            metricDirectory = metricDirectory,
            initiativeMetricType = initiativeMetricType,
        )

        val dataIntegrityViolationException =
            DataIntegrityViolationException(
                "Violation of constraint " +
                    Metadata.ErrorMessages
                        .INITIATIVE_METRIC_VALUE_ALREADY_EXISTS
            )

        every {
            initiativeMetricValueRepository
                .saveAllAndFlush<InitiativeMetricValueEntity>(any())
        } throws dataIntegrityViolationException

        every {
            messageProvider[
                Metadata.ErrorMessages
                    .INITIATIVE_METRIC_VALUE_DUPLICATE
            ]
        } returns "Дубликат метрики {0} для режима {1}"

        // When
        val exception = assertThrows<AiBadRequestException> {
            service.saveInitiativeMetricValue(
                initiativeId = initiativeId,
                request = request,
            )
        }

        // Then
        assertEquals(
            "Дубликат метрики $metricId для режима copilot",
            exception.message,
        )
    }

    @Test
    fun `saveInitiativeMetricValue should rethrow unrelated data integrity violation`() {
        // Given
        val initiativeId = 1L
        val initiativeMetricTypeId = 10L
        val metricId = UUID.randomUUID()
        val periodMonth = currentPeriodMonth()

        val request = createRequest(metricId)
        val metricDirectory = createMetricDirectory(metricId)

        val initiativeMetricType = createInitiativeMetricType(
            initiativeId = initiativeId,
            initiativeMetricTypeId = initiativeMetricTypeId,
            agentType = "copilot",
        )

        stubSuccessfulReadPath(
            initiativeId = initiativeId,
            initiativeMetricTypeId = initiativeMetricTypeId,
            metricId = metricId,
            periodMonth = periodMonth,
            metricDirectory = metricDirectory,
            initiativeMetricType = initiativeMetricType,
        )

        val constraintViolationException = ConstraintViolationException(
            "Another constraint violation",
            SQLException(),
            "some_other_constraint",
        )

        val originalException = DataIntegrityViolationException(
            "Could not save because of another constraint",
            constraintViolationException,
        )

        every {
            initiativeMetricValueRepository
                .saveAllAndFlush<InitiativeMetricValueEntity>(any())
        } throws originalException

        // When
        val actualException =
            assertThrows<DataIntegrityViolationException> {
                service.saveInitiativeMetricValue(
                    initiativeId = initiativeId,
                    request = request,
                )
            }

        // Then
        assertSame(originalException, actualException)
    }

    private fun createRequest(
        metricId: UUID,
        agentType: String = "copilot",
        metricValue: String = "10",
        targetValue: String = "20",
    ): SaveInitiativeMetricValuesRequest {
        return SaveInitiativeMetricValuesRequest(
            metricsValues = listOf(
                SaveInitiativeMetricValueRequest(
                    agentType = agentType,
                    metricId = metricId,
                    metricValue = BigDecimal(metricValue),
                    targetValue = BigDecimal(targetValue),
                )
            )
        )
    }

    private fun createMetricDirectory(
        metricId: UUID,
    ): MetricsDirectoryEntity {
        return MetricsDirectoryEntity().also {
            it.id = metricId
        }
    }

    private fun createInitiativeMetricType(
        initiativeId: Long,
        initiativeMetricTypeId: Long?,
        agentType: String?,
    ): InitiativeMetricTypeEntity {
        return InitiativeMetricTypeEntity(
            aiAgent = AIAgentEntity().also {
                it.id = initiativeId
            },
            agentType = agentType,
        ).also {
            it.id = initiativeMetricTypeId
        }
    }

    private fun stubSuccessfulReadPath(
        initiativeId: Long,
        initiativeMetricTypeId: Long,
        metricId: UUID,
        periodMonth: LocalDate,
        metricDirectory: MetricsDirectoryEntity,
        initiativeMetricType: InitiativeMetricTypeEntity,
        existingValues: List<InitiativeMetricValueEntity> = emptyList(),
    ) {
        every {
            initiativeMetricTypeRepository.existsByAiAgentId(initiativeId)
        } returns true

        every {
            metricsDirectoryRepository.findAllById(setOf(metricId))
        } returns listOf(metricDirectory)

        every {
            initiativeMetricTypeRepository
                .findAllByAiAgentIdAndAgentTypeIn(
                    initiativeId = initiativeId,
                    agentTypes = setOf("copilot"),
                )
        } returns listOf(initiativeMetricType)

        every {
            initiativeMetricValueRepository
                .findAllByInitiativeMetricTypeIdsAndMetricDirectoryIdsAndPeriodMonth(
                    initiativeMetricTypeIds = setOf(
                        initiativeMetricTypeId
                    ),
                    metricDirectoryIds = setOf(metricId),
                    periodMonth = periodMonth,
                )
        } returns existingValues
    }

    private fun stubSaveAllAndFlush():
        MutableList<InitiativeMetricValueEntity> {

        val savedValues =
            mutableListOf<InitiativeMetricValueEntity>()

        every {
            initiativeMetricValueRepository
                .saveAllAndFlush<InitiativeMetricValueEntity>(any())
        } answers {
            firstArg<Iterable<InitiativeMetricValueEntity>>()
                .toList()
                .also { values ->
                    savedValues.addAll(values)
                }
        }

        return savedValues
    }

    private fun currentPeriodMonth(): LocalDate {
        return YearMonth.now().atDay(1)
    }
}
```
