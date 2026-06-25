```java
class InitiativeMetricResponseBuilderTest {

    private lateinit var builder: InitiativeMetricResponseBuilder

    @BeforeEach
    fun setUp() {
        builder = InitiativeMetricResponseBuilder()
    }

    @Test
    fun `historyPeriodFrom should return first day of month two months ago`() {
        // Given
        val expected =
            YearMonth.now()
                .minusMonths(2)
                .atDay(1)

        // When
        val result = builder.historyPeriodFrom()

        // Then
        assertEquals(expected, result)
    }

    @Test
    fun `build should return response only for applicable agent types`() {
        // Given
        val metricId = UUID.randomUUID()

        val metric =
            createMetric(
                id = metricId,
                autonomousApplicability = false,
                copilotApplicability = true,
                requiresAppealsWork = false,
            )

        val copilotMetricType =
            InitiativeMetricTypeEntity(
                aiAgent = createInitiative(initiativeId = 1L),
                agentType = "copilot",
            ).apply {
                id = 10L
            }

        val currentMonth =
            YearMonth.now().atDay(1)

        val metricValue =
            InitiativeMetricValueEntity(
                initiativeMetricType = copilotMetricType,
                metricDirectory = metric,
                periodMonth = currentMonth,
                metricValue = BigDecimal.TEN,
                targetValue = BigDecimal.valueOf(20),
            ).apply {
                id = 100L
            }

        // When
        val result =
            builder.build(
                metrics = listOf(metric),
                requestedAgentTypes = setOf(
                    InitiativeMetricAgentType.AUTONOMOUS,
                    InitiativeMetricAgentType.COPILOT,
                ),
                metricValues = listOf(metricValue),
            )

        // Then
        assertEquals(1, result.size)

        val response = result.first()

        assertEquals(metricId, response.id)
        assertEquals("Метрика", response.name)
        assertEquals("шт", response.unit)
        assertEquals("growth", response.direction)
        assertEquals(setOf("copilot"), response.agentTypes)
        assertEquals(true, response.isActive)
        assertEquals("Описание", response.description)
        assertEquals("monthly", response.frequency)
        assertEquals(BigDecimal.TEN, response.metricValue)
        assertEquals(BigDecimal.valueOf(20), response.targetValue)
    }

    @Test
    fun `build should return separate responses for same metric with different agent types`() {
        // Given
        val metricId = UUID.randomUUID()
        val initiative = createInitiative(initiativeId = 1L)

        val metric =
            createMetric(
                id = metricId,
                autonomousApplicability = true,
                copilotApplicability = true,
            )

        val autonomousMetricType =
            InitiativeMetricTypeEntity(
                aiAgent = initiative,
                agentType = "autonomous",
            ).apply {
                id = 10L
            }

        val copilotMetricType =
            InitiativeMetricTypeEntity(
                aiAgent = initiative,
                agentType = "copilot",
            ).apply {
                id = 20L
            }

        val autonomousValue =
            InitiativeMetricValueEntity(
                initiativeMetricType = autonomousMetricType,
                metricDirectory = metric,
                periodMonth = YearMonth.now().atDay(1),
                metricValue = BigDecimal.valueOf(100),
                targetValue = BigDecimal.valueOf(200),
            ).apply {
                id = 100L
            }

        val copilotValue =
            InitiativeMetricValueEntity(
                initiativeMetricType = copilotMetricType,
                metricDirectory = metric,
                periodMonth = YearMonth.now().atDay(1),
                metricValue = BigDecimal.valueOf(300),
                targetValue = BigDecimal.valueOf(400),
            ).apply {
                id = 200L
            }

        // When
        val result =
            builder.build(
                metrics = listOf(metric),
                requestedAgentTypes = setOf(
                    InitiativeMetricAgentType.AUTONOMOUS,
                    InitiativeMetricAgentType.COPILOT,
                ),
                metricValues = listOf(autonomousValue, copilotValue),
            )

        // Then
        assertEquals(2, result.size)

        val autonomousResponse =
            result.first { response ->
                response.agentTypes == setOf("autonomous")
            }

        val copilotResponse =
            result.first { response ->
                response.agentTypes == setOf("copilot")
            }

        assertEquals(BigDecimal.valueOf(100), autonomousResponse.metricValue)
        assertEquals(BigDecimal.valueOf(200), autonomousResponse.targetValue)

        assertEquals(BigDecimal.valueOf(300), copilotResponse.metricValue)
        assertEquals(BigDecimal.valueOf(400), copilotResponse.targetValue)
    }

    @Test
    fun `build should use latest period value as main metric value and target value`() {
        // Given
        val metricId = UUID.randomUUID()
        val initiative = createInitiative(initiativeId = 1L)

        val metric =
            createMetric(
                id = metricId,
                copilotApplicability = true,
            )

        val metricType =
            InitiativeMetricTypeEntity(
                aiAgent = initiative,
                agentType = "copilot",
            ).apply {
                id = 10L
            }

        val previousMonthValue =
            InitiativeMetricValueEntity(
                initiativeMetricType = metricType,
                metricDirectory = metric,
                periodMonth = YearMonth.now().minusMonths(1).atDay(1),
                metricValue = BigDecimal.valueOf(10),
                targetValue = BigDecimal.valueOf(20),
            ).apply {
                id = 100L
            }

        val currentMonthValue =
            InitiativeMetricValueEntity(
                initiativeMetricType = metricType,
                metricDirectory = metric,
                periodMonth = YearMonth.now().atDay(1),
                metricValue = BigDecimal.valueOf(30),
                targetValue = BigDecimal.valueOf(40),
            ).apply {
                id = 200L
            }

        // When
        val result =
            builder.build(
                metrics = listOf(metric),
                requestedAgentTypes = setOf(InitiativeMetricAgentType.COPILOT),
                metricValues = listOf(previousMonthValue, currentMonthValue),
            )

        // Then
        val response = result.first()

        assertEquals(BigDecimal.valueOf(30), response.metricValue)
        assertEquals(BigDecimal.valueOf(40), response.targetValue)
    }

    @Test
    fun `build should build periods with correct indexes`() {
        // Given
        val metricId = UUID.randomUUID()
        val initiative = createInitiative(initiativeId = 1L)

        val metric =
            createMetric(
                id = metricId,
                copilotApplicability = true,
            )

        val metricType =
            InitiativeMetricTypeEntity(
                aiAgent = initiative,
                agentType = "copilot",
            ).apply {
                id = 10L
            }

        val currentMonth =
            YearMonth.now()

        val oldestHistoryMonth =
            currentMonth.minusMonths(2)

        val middleHistoryMonth =
            currentMonth.minusMonths(1)

        val currentMonthValue =
            InitiativeMetricValueEntity(
                initiativeMetricType = metricType,
                metricDirectory = metric,
                periodMonth = currentMonth.atDay(1),
                metricValue = BigDecimal.valueOf(30),
                targetValue = BigDecimal.valueOf(40),
            ).apply {
                id = 300L
            }

        val oldestHistoryValue =
            InitiativeMetricValueEntity(
                initiativeMetricType = metricType,
                metricDirectory = metric,
                periodMonth = oldestHistoryMonth.atDay(1),
                metricValue = BigDecimal.valueOf(10),
                targetValue = BigDecimal.valueOf(20),
            ).apply {
                id = 100L
            }

        val middleHistoryValue =
            InitiativeMetricValueEntity(
                initiativeMetricType = metricType,
                metricDirectory = metric,
                periodMonth = middleHistoryMonth.atDay(1),
                metricValue = BigDecimal.valueOf(20),
                targetValue = BigDecimal.valueOf(30),
            ).apply {
                id = 200L
            }

        // When
        val result =
            builder.build(
                metrics = listOf(metric),
                requestedAgentTypes = setOf(InitiativeMetricAgentType.COPILOT),
                metricValues = listOf(
                    currentMonthValue,
                    oldestHistoryValue,
                    middleHistoryValue,
                ),
            )

        // Then
        val periods = result.first().periods

        assertEquals(3, periods.size)

        assertEquals(0, periods[0].index)
        assertEquals(BigDecimal.valueOf(30), periods[0].value)
        assertEquals(formatPeriod(currentMonth), periods[0].period)

        assertEquals(1, periods[1].index)
        assertEquals(BigDecimal.valueOf(10), periods[1].value)
        assertEquals(formatPeriod(oldestHistoryMonth), periods[1].period)

        assertEquals(2, periods[2].index)
        assertEquals(BigDecimal.valueOf(20), periods[2].value)
        assertEquals(formatPeriod(middleHistoryMonth), periods[2].period)
    }

    @Test
    fun `build should ignore periods outside last three months`() {
        // Given
        val metricId = UUID.randomUUID()
        val initiative = createInitiative(initiativeId = 1L)

        val metric =
            createMetric(
                id = metricId,
                copilotApplicability = true,
            )

        val metricType =
            InitiativeMetricTypeEntity(
                aiAgent = initiative,
                agentType = "copilot",
            ).apply {
                id = 10L
            }

        val oldValue =
            InitiativeMetricValueEntity(
                initiativeMetricType = metricType,
                metricDirectory = metric,
                periodMonth = YearMonth.now().minusMonths(3).atDay(1),
                metricValue = BigDecimal.valueOf(999),
                targetValue = BigDecimal.valueOf(999),
            ).apply {
                id = 100L
            }

        val currentValue =
            InitiativeMetricValueEntity(
                initiativeMetricType = metricType,
                metricDirectory = metric,
                periodMonth = YearMonth.now().atDay(1),
                metricValue = BigDecimal.TEN,
                targetValue = BigDecimal.valueOf(20),
            ).apply {
                id = 200L
            }

        // When
        val result =
            builder.build(
                metrics = listOf(metric),
                requestedAgentTypes = setOf(InitiativeMetricAgentType.COPILOT),
                metricValues = listOf(oldValue, currentValue),
            )

        // Then
        val periods = result.first().periods

        assertEquals(1, periods.size)
        assertEquals(0, periods.first().index)
        assertEquals(BigDecimal.TEN, periods.first().value)
    }

    private fun createInitiative(
        initiativeId: Long,
    ): AIAgentEntity {
        return AIAgentEntity().apply {
            id = initiativeId
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

    private fun formatPeriod(
        yearMonth: YearMonth,
    ): String {
        return yearMonth.format(
            DateTimeFormatter.ofPattern("LLLL yyyy", Locale("ru"))
        )
    }
}
```
