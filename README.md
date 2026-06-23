```java
@WebMvcTest(AIAgentInitiativeController::class)
internal class AIAgentInitiativeControllerTest {

    @Autowired
    private lateinit var mockMvc: MockMvc

    @Autowired
    private lateinit var objectMapper: ObjectMapper

    @MockkBean
    private lateinit var aiAgentInitiativeService: AIAgentInitiativeService

    @Test
    @WithMockUser(authorities = ["PROJECT_OFFICE"])
    fun `saveInitiativeMetricValue should return success response`() {
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
            aiAgentInitiativeService.saveInitiativeMetricValue(
                initiativeId = initiativeId,
                request = request,
            )
        } returns SaveInitiativeMetricValueResponse.success()

        // When & Then
        mockMvc.post("/api/v1/ai-agent/initiatives/{initiativeId}/metrics/value", initiativeId) {
            contentType = MediaType.APPLICATION_JSON
            content = objectMapper.writeValueAsString(request)
        }
            .andExpect {
                status { isOk() }
                jsonPath("$.code") { value(0) }
                jsonPath("$.message") { value("Metrics saved") }
            }

        verify(exactly = 1) {
            aiAgentInitiativeService.saveInitiativeMetricValue(
                initiativeId = initiativeId,
                request = request,
            )
        }
    }
}


```
