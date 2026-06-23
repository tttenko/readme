```java
@Import(value = [AIAgentInitiativeController::class])
@ContextConfiguration(classes = [AIAgentInitiativeControllerTest.Config::class])
internal class AIAgentInitiativeControllerTest : ControllerTestBase() {

    @Autowired
    private lateinit var service: AIAgentInitiativeService

    @Autowired
    private lateinit var mapper: ObjectMapper

    @TestConfiguration
    internal class Config {

        @Bean
        open fun service() = mockk<AIAgentInitiativeService>()
    }

    @BeforeEach
    fun setUp() {
        clearMocks(service)
    }

    @Test
    fun `should save initiative metric value with success`() {
        // given
        val initiativeId = 1L
        val metricId = UUID.randomUUID()

        val request = SaveInitiativeMetricValueRequest(
            agentType = "copilot",
            metricId = metricId,
            metricValue = BigDecimal("10"),
            targetValue = BigDecimal("20"),
        )

        val expectedResult = SaveInitiativeMetricValueResponse(
            code = 0,
            message = "Metrics saved",
        )

        every {
            service.saveInitiativeMetricValue(
                initiativeId = initiativeId,
                request = request,
            )
        } returns expectedResult

        // when / then
        mockMvc.perform(
            MockMvcRequestBuilders.post(API_URL, initiativeId)
                .contentType(MediaType.APPLICATION_JSON)
                .content(mapper.writeValueAsString(request))
        )
            .andExpect(MockMvcResultMatchers.status().isOk)
            .andExpect(MockMvcResultMatchers.jsonPath("$.code", equalTo(0)))
            .andExpect(MockMvcResultMatchers.jsonPath("$.message", equalTo("Metrics saved")))

        verify(exactly = 1) {
            service.saveInitiativeMetricValue(
                initiativeId = initiativeId,
                request = request,
            )
        }
    }

    @Test
    fun `should return bad request when agent type is missing`() {
        // given
        val initiativeId = 1L
        val metricId = UUID.randomUUID()

        val request = """
            {
              "id": "$metricId",
              "metricValue": 10,
              "targetValue": 20
            }
        """.trimIndent()

        // when / then
        mockMvc.perform(
            MockMvcRequestBuilders.post(API_URL, initiativeId)
                .contentType(MediaType.APPLICATION_JSON)
                .content(request)
        )
            .andExpect(MockMvcResultMatchers.status().isBadRequest)

        verify(exactly = 0) {
            service.saveInitiativeMetricValue(
                initiativeId = any(),
                request = any(),
            )
        }
    }

    @Test
    fun `should return bad request when metric id is missing`() {
        // given
        val initiativeId = 1L

        val request = """
            {
              "agentType": "copilot",
              "metricValue": 10,
              "targetValue": 20
            }
        """.trimIndent()

        // when / then
        mockMvc.perform(
            MockMvcRequestBuilders.post(API_URL, initiativeId)
                .contentType(MediaType.APPLICATION_JSON)
                .content(request)
        )
            .andExpect(MockMvcResultMatchers.status().isBadRequest)

        verify(exactly = 0) {
            service.saveInitiativeMetricValue(
                initiativeId = any(),
                request = any(),
            )
        }
    }

    companion object {
        private const val API_URL = "/api/v1/ai-agent/initiatives/{initiativeId}/metrics/value"
    }
}

```
