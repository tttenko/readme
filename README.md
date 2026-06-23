```java
@MockK
private lateinit var metricsDirectoryRepository: MetricsDirectoryRepository

@MockK
private lateinit var initiativeMetricTypeRepository: InitiativeMetricTypeRepository

@MockK
private lateinit var initiativeMetricValueRepository: InitiativeMetricValueRepository

@Test
fun `saveInitiativeMetricValue should create metric value when value does not exist`() {
    // Given
    val initiativeId = 1L
    val initiativeMetricTypeId = 10L
    val metricId = UUID.randomUUID()

    val request = SaveInitiativeMetricValueRequest(
        agentType = "copilot",
        metricId = metricId,
        metricValue = BigDecimal("10"),
        targetValue = BigDecimal("20"),
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

    val savedMetricValueSlot = slot<InitiativeMetricValueEntity>()

    every {
        metricsDirectoryRepository.findById(metricId)
    } returns Optional.of(metricDirectory)

    every {
        initiativeMetricTypeRepository.existsByAiAgentId(initiativeId)
    } returns true

    every {
        initiativeMetricTypeRepository.findByAiAgentIdAndAgentType(
            initiativeId = initiativeId,
            agentType = "copilot",
        )
    } returns initiativeMetricType

    every {
        initiativeMetricValueRepository.findByInitiativeMetricTypeIdAndMetricDirectoryId(
            initiativeMetricTypeId = initiativeMetricTypeId,
            metricDirectoryId = metricId,
        )
    } returns null

    every {
        initiativeMetricValueRepository.save(capture(savedMetricValueSlot))
    } answers {
        savedMetricValueSlot.captured
    }

    // When
    val response = service.saveInitiativeMetricValue(
        initiativeId = initiativeId,
        request = request,
    )

    // Then
    assertEquals(0, response.code)
    assertEquals("Metrics saved", response.message)

    assertSame(
        initiativeMetricType,
        savedMetricValueSlot.captured.initiativeMetricType,
    )
    assertSame(
        metricDirectory,
        savedMetricValueSlot.captured.metricDirectory,
    )
    assertEquals(
        BigDecimal("10"),
        savedMetricValueSlot.captured.metricValue,
    )
    assertEquals(
        BigDecimal("20"),
        savedMetricValueSlot.captured.targetValue,
    )
}
3. Тест: обновление существующей записи
@Test
fun `saveInitiativeMetricValue should update metric value when value already exists`() {
    // Given
    val initiativeId = 1L
    val initiativeMetricTypeId = 10L
    val metricId = UUID.randomUUID()

    val request = SaveInitiativeMetricValueRequest(
        agentType = "copilot",
        metricId = metricId,
        metricValue = BigDecimal("15"),
        targetValue = BigDecimal("25"),
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

    val savedMetricValueSlot = slot<InitiativeMetricValueEntity>()

    every {
        metricsDirectoryRepository.findById(metricId)
    } returns Optional.of(metricDirectory)

    every {
        initiativeMetricTypeRepository.existsByAiAgentId(initiativeId)
    } returns true

    every {
        initiativeMetricTypeRepository.findByAiAgentIdAndAgentType(
            initiativeId = initiativeId,
            agentType = "copilot",
        )
    } returns initiativeMetricType

    every {
        initiativeMetricValueRepository.findByInitiativeMetricTypeIdAndMetricDirectoryId(
            initiativeMetricTypeId = initiativeMetricTypeId,
            metricDirectoryId = metricId,
        )
    } returns existingMetricValue

    every {
        initiativeMetricValueRepository.save(capture(savedMetricValueSlot))
    } answers {
        savedMetricValueSlot.captured
    }

    // When
    val response = service.saveInitiativeMetricValue(
        initiativeId = initiativeId,
        request = request,
    )

    // Then
    assertEquals(0, response.code)
    assertEquals("Metrics saved", response.message)

    assertSame(
        existingMetricValue,
        savedMetricValueSlot.captured,
    )
    assertEquals(
        BigDecimal("15"),
        existingMetricValue.metricValue,
    )
    assertEquals(
        BigDecimal("25"),
        existingMetricValue.targetValue,
    )
}
4. Тест: ошибка, если передан неверный agentType
@Test
fun `saveInitiativeMetricValue should throw bad request when agent type is wrong`() {
    // Given
    val request = SaveInitiativeMetricValueRequest(
        agentType = "wrong",
        metricId = UUID.randomUUID(),
        metricValue = BigDecimal("10"),
        targetValue = BigDecimal("20"),
    )

    every {
        messageProvider[WRONG_INITIATIVE_METRIC_AGENT_TYPE]
    } returns "Недопустимый режим работы инициативы: {0}"

    // When & Then
    assertThrows<AiBadRequestException> {
        service.saveInitiativeMetricValue(
            initiativeId = 1L,
            request = request,
        )
    }
}
5. Тест: ошибка, если метрика не найдена
@Test
fun `saveInitiativeMetricValue should throw bad request when metric not found`() {
    // Given
    val initiativeId = 1L
    val metricId = UUID.randomUUID()

    val request = SaveInitiativeMetricValueRequest(
        agentType = "copilot",
        metricId = metricId,
        metricValue = BigDecimal("10"),
        targetValue = BigDecimal("20"),
    )

    every {
        metricsDirectoryRepository.findById(metricId)
    } returns Optional.empty()

    every {
        messageProvider[INITIATIVE_METRIC_NOT_FOUND]
    } returns "Метрика с идентификатором {0} не найдена"

    // When & Then
    assertThrows<AiBadRequestException> {
        service.saveInitiativeMetricValue(
            initiativeId = initiativeId,
            request = request,
        )
    }
}
6. Тест: ошибка 409, если у инициативы нет режимов работы
@Test
fun `saveInitiativeMetricValue should throw conflict when initiative has no metric types`() {
    // Given
    val initiativeId = 1L
    val metricId = UUID.randomUUID()

    val request = SaveInitiativeMetricValueRequest(
        agentType = "copilot",
        metricId = metricId,
        metricValue = BigDecimal("10"),
        targetValue = BigDecimal("20"),
    )

    val metricDirectory = MetricsDirectoryEntity().also {
        it.id = metricId
    }

    every {
        metricsDirectoryRepository.findById(metricId)
    } returns Optional.of(metricDirectory)

    every {
        initiativeMetricTypeRepository.existsByAiAgentId(initiativeId)
    } returns false

    every {
        messageProvider[INITIATIVE_METRIC_TYPES_NOT_FOUND]
    } returns "Для инициативы с идентификатором {0} не найдены режимы работы"

    // When & Then
    assertThrows<AiConflictException> {
        service.saveInitiativeMetricValue(
            initiativeId = initiativeId,
            request = request,
        )
    }
}
7. Тест: ошибка 409, если у инициативы нет конкретного agentType
@Test
fun `saveInitiativeMetricValue should throw conflict when initiative agent type not found`() {
    // Given
    val initiativeId = 1L
    val metricId = UUID.randomUUID()

    val request = SaveInitiativeMetricValueRequest(
        agentType = "copilot",
        metricId = metricId,
        metricValue = BigDecimal("10"),
        targetValue = BigDecimal("20"),
    )

    val metricDirectory = MetricsDirectoryEntity().also {
        it.id = metricId
    }

    every {
        metricsDirectoryRepository.findById(metricId)
    } returns Optional.of(metricDirectory)

    every {
        initiativeMetricTypeRepository.existsByAiAgentId(initiativeId)
    } returns true

    every {
        initiativeMetricTypeRepository.findByAiAgentIdAndAgentType(
            initiativeId = initiativeId,
            agentType = "copilot",
        )
    } returns null

    every {
        messageProvider[INITIATIVE_METRIC_AGENT_TYPE_NOT_FOUND]
    } returns "Для инициативы с идентификатором {0} не найден режим работы: {1}"

    // When & Then
    assertThrows<AiConflictException> {
        service.saveInitiativeMetricValue(
            initiativeId = initiativeId,
            request = request,
        )
    }
}
```
