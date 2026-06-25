```java
 @Test
    fun `createInitiativeAgentTypes should delegate to initiativeAgentTypesCreator and return response`() {
        // Given
        val initiativeId = 1L

        val request =
            UpdateInitiativeAgentTypesRequest(
                agentTypes = setOf("autonomous", "copilot")
            )

        val expectedResponse =
            listOf(
                InitiativeMetricResponse(
                    id = UUID.randomUUID(),
                    name = "Метрика",
                    unit = "шт",
                    direction = "growth",
                    agentTypes = setOf("autonomous", "copilot"),
                    isActive = true,
                    description = "Описание",
                    frequency = "monthly",
                    metricValue = null,
                    targetValue = null,
                    periods = emptyList(),
                )
            )

        every {
            initiativeAgentTypesCreator.createInitiativeAgentTypes(
                initiativeId = initiativeId,
                request = request,
            )
        } returns expectedResponse

        // When
        val actualResponse =
            service.createInitiativeAgentTypes(
                initiativeId = initiativeId,
                request = request,
            )

        // Then
        assertEquals(expectedResponse, actualResponse)

        verify(exactly = 1) {
            initiativeAgentTypesCreator.createInitiativeAgentTypes(
                initiativeId = initiativeId,
                request = request,
            )
        }

        verify(exactly = 0) {
            initiativeMetricValueSaver.saveInitiativeMetricValue(any(), any())
        }

        verify(exactly = 0) {
            initiativeAgentTypesUpdater.update(any(), any())
        }

        confirmVerified(
            initiativeMetricValueSaver,
            initiativeAgentTypesUpdater,
            initiativeAgentTypesCreator,
        )
    }
}

@Test
fun `buildWithoutValues should build responses with null metric values and empty periods`() {
    // Given
    val metricId = UUID.randomUUID()

    val metric =
        createMetric(
            id = metricId,
            autonomousApplicability = true,
            copilotApplicability = true,
            requiresAppealsWork = false,
        )

    val requestedAgentTypes =
        setOf(
            InitiativeMetricAgentType.AUTONOMOUS,
            InitiativeMetricAgentType.COPILOT,
        )

    // When
    val result =
        builder.buildWithoutValues(
            metrics = listOf(metric),
            requestedAgentTypes = requestedAgentTypes,
        )

    // Then
    assertEquals(1, result.size)

    val response = result.first()

    assertEquals(metricId, response.id)
    assertEquals("Метрика", response.name)
    assertEquals("шт", response.unit)
    assertEquals("growth", response.direction)
    assertEquals(setOf("autonomous", "copilot"), response.agentTypes)
    assertEquals(true, response.isActive)
    assertEquals("Описание", response.description)
    assertEquals("monthly", response.frequency)

    assertNull(response.metricValue)
    assertNull(response.targetValue)
    assertTrue(response.periods.isEmpty())
}
Тест на пустой список метрик
@Test
fun `buildWithoutValues should return empty list when metrics are empty`() {
    // Given
    val requestedAgentTypes =
        setOf(
            InitiativeMetricAgentType.AUTONOMOUS,
            InitiativeMetricAgentType.COPILOT,
        )

    // When
    val result =
        builder.buildWithoutValues(
            metrics = emptyList(),
            requestedAgentTypes = requestedAgentTypes,
        )

    // Then
    assertTrue(result.isEmpty())
}
Тест на несколько метрик
@Test
fun `buildWithoutValues should build response for each metric`() {
    // Given
    val firstMetric =
        createMetric(
            id = UUID.randomUUID(),
            autonomousApplicability = true,
        )

    val secondMetric =
        createMetric(
            id = UUID.randomUUID(),
            copilotApplicability = true,
        )

    val requestedAgentTypes =
        setOf(
            InitiativeMetricAgentType.AUTONOMOUS,
            InitiativeMetricAgentType.COPILOT,
        )

    // When
    val result =
        builder.buildWithoutValues(
            metrics = listOf(firstMetric, secondMetric),
            requestedAgentTypes = requestedAgentTypes,
        )

    // Then
    assertEquals(2, result.size)

    result.forEach { response ->
        assertEquals(setOf("autonomous", "copilot"), response.agentTypes)
        assertNull(response.metricValue)
        assertNull(response.targetValue)
        assertTrue(response.periods.isEmpty())
    }

    assertEquals(firstMetric.id, result[0].id)
    assertEquals(secondMetric.id, result[1].id)
}

```
