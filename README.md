```java
@ExtendWith(MockKExtension::class)
class AIAgentInitiativeServiceTest {

    @MockK
    private lateinit var initiativeMetricValueSaver: InitiativeMetricValueSaver

    @MockK
    private lateinit var initiativeAgentTypesUpdater: InitiativeAgentTypesUpdater

    private lateinit var service: AIAgentInitiativeService

    @BeforeEach
    fun setUp() {
        service = AIAgentInitiativeService(
            initiativeMetricValueSaver = initiativeMetricValueSaver,
            initiativeAgentTypesUpdater = initiativeAgentTypesUpdater,
        )
    }

    @Test
    fun `saveInitiativeMetricValue should delegate to initiativeMetricValueSaver and return response`() {
        // Given
        val initiativeId = 1L

        val request =
            SaveInitiativeMetricValueRequest(
                agentType = "copilot",
                metricId = UUID.randomUUID(),
                metricValue = BigDecimal.TEN,
                targetValue = BigDecimal.valueOf(20),
            )

        val expectedResponse =
            SaveInitiativeMetricValueResponse.success()

        every {
            initiativeMetricValueSaver.saveInitiativeMetricValue(
                initiativeId = initiativeId,
                request = request,
            )
        } returns expectedResponse

        // When
        val actualResponse =
            service.saveInitiativeMetricValue(
                initiativeId = initiativeId,
                request = request,
            )

        // Then
        assertEquals(expectedResponse, actualResponse)

        verify(exactly = 1) {
            initiativeMetricValueSaver.saveInitiativeMetricValue(
                initiativeId = initiativeId,
                request = request,
            )
        }

        verify(exactly = 0) {
            initiativeAgentTypesUpdater.update(any(), any())
        }

        confirmVerified(
            initiativeMetricValueSaver,
            initiativeAgentTypesUpdater,
        )
    }

    @Test
    fun `updateInitiativeAgentTypes should delegate to initiativeAgentTypesUpdater and return response`() {
        // Given
        val initiativeId = 1L

        val request =
            UpdateInitiativeAgentTypesRequest(
                agentTypes = setOf("copilot", "autonomous")
            )

        val expectedResponse =
            listOf(
                InitiativeMetricResponse(
                    id = UUID.randomUUID(),
                    name = "Метрика",
                    unit = "шт",
                    direction = "growth",
                    agentTypes = setOf("copilot"),
                    isActive = true,
                    description = "Описание",
                    frequency = "monthly",
                    metricValue = BigDecimal.TEN,
                    targetValue = BigDecimal.valueOf(20),
                    periods = emptyList(),
                )
            )

        every {
            initiativeAgentTypesUpdater.update(
                initiativeId = initiativeId,
                request = request,
            )
        } returns expectedResponse

        // When
        val actualResponse =
            service.updateInitiativeAgentTypes(
                initiativeId = initiativeId,
                request = request,
            )

        // Then
        assertEquals(expectedResponse, actualResponse)

        verify(exactly = 1) {
            initiativeAgentTypesUpdater.update(
                initiativeId = initiativeId,
                request = request,
            )
        }

        verify(exactly = 0) {
            initiativeMetricValueSaver.saveInitiativeMetricValue(any(), any())
        }

        confirmVerified(
            initiativeMetricValueSaver,
            initiativeAgentTypesUpdater,
        )
    }
}
```
