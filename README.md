```java
@Import(value = [MetricsController::class])
@ContextConfiguration(classes = [MetricsControllerTest.Config::class])
internal class MetricsControllerTest : ControllerTestBase() {

    @Autowired
    private lateinit var metricsService: MetricsService

    @Autowired
    private lateinit var mapper: ObjectMapper

    @TestConfiguration
    internal class Config {

        @Bean
        open fun metricsService() = mockk<MetricsService>()
    }

    @BeforeEach
    fun setup() {
        clearMocks(firstMock = metricsService)
    }

    @Test
    fun `should get metrics with success`() {
        // given
        val metricId = UUID.randomUUID()
        val lastModifiedAt = LocalDateTime.of(2026, 6, 19, 12, 30)

        val expectedResult = listOf(
            CreateMetricResponse(
                id = metricId,
                name = "Количество обращений",
                unit = "шт.",
                direction = "Рост",
                agentTypes = setOf(
                    MetricAgentType.AUTONOMOUS,
                    MetricAgentType.COPILOT,
                ),
                isActive = true,
                description = "Описание метрики",
                frequency = "Ежемесячно",
                canBeDeleted = true,
                lastModifiedAt = lastModifiedAt,
                lastModifiedBy = "Иванов Иван Иванович",
            )
        )

        every {
            metricsService.getMetrics()
        } returns expectedResult

        // when / then
        mockMvc.perform(
            MockMvcRequestBuilders.get(API_URL)
        )
            .andExpect(MockMvcResultMatchers.status().isOk)
            .andExpect(
                MockMvcResultMatchers.jsonPath(
                    "$[0].id",
                    equalTo(metricId.toString())
                )
            )
            .andExpect(
                MockMvcResultMatchers.jsonPath(
                    "$[0].name",
                    equalTo("Количество обращений")
                )
            )
            .andExpect(
                MockMvcResultMatchers.jsonPath(
                    "$[0].unit",
                    equalTo("шт.")
                )
            )
            .andExpect(
                MockMvcResultMatchers.jsonPath(
                    "$[0].direction",
                    equalTo("Рост")
                )
            )
            .andExpect(
                MockMvcResultMatchers.jsonPath(
                    "$[0].agentTypes[0]",
                    equalTo(MetricAgentType.AUTONOMOUS)
                )
            )
            .andExpect(
                MockMvcResultMatchers.jsonPath(
                    "$[0].isActive",
                    equalTo(true)
                )
            )
            .andExpect(
                MockMvcResultMatchers.jsonPath(
                    "$[0].description",
                    equalTo("Описание метрики")
                )
            )
            .andExpect(
                MockMvcResultMatchers.jsonPath(
                    "$[0].frequency",
                    equalTo("Ежемесячно")
                )
            )
            .andExpect(
                MockMvcResultMatchers.jsonPath(
                    "$[0].canBeDeleted",
                    equalTo(true)
                )
            )
            .andExpect(
                MockMvcResultMatchers.jsonPath(
                    "$[0].lastModifiedBy",
                    equalTo("Иванов Иван Иванович")
                )
            )

        verify(exactly = 1) {
            metricsService.getMetrics()
        }
    }

    @Test
    fun `should get empty metrics with success`() {
        // given
        every {
            metricsService.getMetrics()
        } returns emptyList()

        // when / then
        mockMvc.perform(
            MockMvcRequestBuilders.get(API_URL)
        )
            .andExpect(MockMvcResultMatchers.status().isOk)
            .andExpect(
                MockMvcResultMatchers.content().json("[]")
            )

        verify(exactly = 1) {
            metricsService.getMetrics()
        }
    }

    @Test
    fun `should create metric with success`() {
        // given
        val metricId = UUID.randomUUID()
        val lastModifiedAt = LocalDateTime.of(2026, 6, 19, 12, 30)

        val request = CreateMetricRequest(
            name = "Количество обращений",
            unit = "шт.",
            direction = "Рост",
            agentTypes = setOf(
                MetricAgentType.AUTONOMOUS,
                MetricAgentType.COPILOT,
            ),
            description = "Описание метрики",
            frequency = "Ежемесячно",
        )

        val expectedResult = CreateMetricResponse(
            id = metricId,
            name = "Количество обращений",
            unit = "шт.",
            direction = "Рост",
            agentTypes = setOf(
                MetricAgentType.AUTONOMOUS,
                MetricAgentType.COPILOT,
            ),
            isActive = true,
            description = "Описание метрики",
            frequency = "Ежемесячно",
            canBeDeleted = true,
            lastModifiedAt = lastModifiedAt,
            lastModifiedBy = "Иванов Иван Иванович",
        )

        every {
            metricsService.createMetric(request = request)
        } returns expectedResult

        // when / then
        mockMvc.perform(
            MockMvcRequestBuilders.post(API_URL)
                .contentType(MediaType.APPLICATION_JSON)
                .content(
                    mapper.writeValueAsString(request)
                )
        )
            .andExpect(MockMvcResultMatchers.status().isCreated)
            .andExpect(
                MockMvcResultMatchers.jsonPath(
                    "$.id",
                    equalTo(metricId.toString())
                )
            )
            .andExpect(
                MockMvcResultMatchers.jsonPath(
                    "$.name",
                    equalTo("Количество обращений")
                )
            )
            .andExpect(
                MockMvcResultMatchers.jsonPath(
                    "$.isActive",
                    equalTo(true)
                )
            )
            .andExpect(
                MockMvcResultMatchers.jsonPath(
                    "$.canBeDeleted",
                    equalTo(true)
                )
            )
            .andExpect(
                MockMvcResultMatchers.jsonPath(
                    "$.lastModifiedBy",
                    equalTo("Иванов Иван Иванович")
                )
            )

        verify(exactly = 1) {
            metricsService.createMetric(request = request)
        }
    }

    @Test
    fun `should update metric activity with success`() {
        // given
        val metricId = UUID.randomUUID()
        val lastModifiedAt = LocalDateTime.of(2026, 6, 19, 12, 30)

        val request = UpdateMetricActivityRequest(
            active = false,
        )

        val expectedResult = CreateMetricResponse(
            id = metricId,
            name = "Количество обращений",
            unit = "шт.",
            direction = "Рост",
            agentTypes = setOf(
                MetricAgentType.AUTONOMOUS,
            ),
            isActive = false,
            description = "Описание метрики",
            frequency = "Ежемесячно",
            canBeDeleted = false,
            lastModifiedAt = lastModifiedAt,
            lastModifiedBy = "Иванов Иван Иванович",
        )

        every {
            metricsService.updateMetricActivity(
                metricId = metricId,
                request = request,
            )
        } returns expectedResult

        // when / then
        mockMvc.perform(
            MockMvcRequestBuilders.patch("$API_URL/$metricId")
                .contentType(MediaType.APPLICATION_JSON)
                .content(
                    mapper.writeValueAsString(request)
                )
        )
            .andExpect(MockMvcResultMatchers.status().isOk)
            .andExpect(
                MockMvcResultMatchers.jsonPath(
                    "$.id",
                    equalTo(metricId.toString())
                )
            )
            .andExpect(
                MockMvcResultMatchers.jsonPath(
                    "$.isActive",
                    equalTo(false)
                )
            )
            .andExpect(
                MockMvcResultMatchers.jsonPath(
                    "$.canBeDeleted",
                    equalTo(false)
                )
            )
            .andExpect(
                MockMvcResultMatchers.jsonPath(
                    "$.lastModifiedBy",
                    equalTo("Иванов Иван Иванович")
                )
            )

        verify(exactly = 1) {
            metricsService.updateMetricActivity(
                metricId = metricId,
                request = request,
            )
        }
    }

    @Test
    fun `should update metric with success`() {
        // given
        val metricId = UUID.randomUUID()
        val lastModifiedAt = LocalDateTime.of(2026, 6, 19, 12, 30)

        val request = UpdateMetricRequest(
            name = "Количество успешных обращений",
            unit = "шт.",
            direction = "Рост",
            agentTypes = setOf(
                MetricAgentType.AUTONOMOUS,
                MetricAgentType.COPILOT,
                MetricAgentType.APPEALS,
            ),
            description = "Новое описание метрики",
            frequency = "Ежемесячно",
        )

        val expectedResult = CreateMetricResponse(
            id = metricId,
            name = "Количество успешных обращений",
            unit = "шт.",
            direction = "Рост",
            agentTypes = setOf(
                MetricAgentType.AUTONOMOUS,
                MetricAgentType.COPILOT,
                MetricAgentType.APPEALS,
            ),
            isActive = true,
            description = "Новое описание метрики",
            frequency = "Ежемесячно",
            canBeDeleted = true,
            lastModifiedAt = lastModifiedAt,
            lastModifiedBy = "Иванов Иван Иванович",
        )

        every {
            metricsService.updateMetric(
                metricId = metricId,
                request = request,
            )
        } returns expectedResult

        // when / then
        mockMvc.perform(
            MockMvcRequestBuilders.put("$API_URL/$metricId")
                .contentType(MediaType.APPLICATION_JSON)
                .content(
                    mapper.writeValueAsString(request)
                )
        )
            .andExpect(MockMvcResultMatchers.status().isOk)
            .andExpect(
                MockMvcResultMatchers.jsonPath(
                    "$.id",
                    equalTo(metricId.toString())
                )
            )
            .andExpect(
                MockMvcResultMatchers.jsonPath(
                    "$.name",
                    equalTo("Количество успешных обращений")
                )
            )
            .andExpect(
                MockMvcResultMatchers.jsonPath(
                    "$.agentTypes.length()",
                    equalTo(3)
                )
            )
            .andExpect(
                MockMvcResultMatchers.jsonPath(
                    "$.isActive",
                    equalTo(true)
                )
            )
            .andExpect(
                MockMvcResultMatchers.jsonPath(
                    "$.canBeDeleted",
                    equalTo(true)
                )
            )

        verify(exactly = 1) {
            metricsService.updateMetric(
                metricId = metricId,
                request = request,
            )
        }
    }

    private companion object {
        private const val API_URL = "/api/v1/reference/metrics"
    }
}
```
