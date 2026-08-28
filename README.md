```java

@ExtendWith(MockKExtension::class)
class InitiativeMetricResponseBuilderTest {

    private lateinit var builder: InitiativeMetricResponseBuilder
    private lateinit var metricApplicabilityPolicy: InitiativeMetricApplicabilityPolicy

    @BeforeEach
    fun setUp() {
        metricApplicabilityPolicy = InitiativeMetricApplicabilityPolicy()

        builder = InitiativeMetricResponseBuilder(
            metricApplicabilityPolicy = metricApplicabilityPolicy,
            planExecutionCalculator = InitiativeMetricPlanExecutionCalculator(),
        )
    }

    @Test
    fun `build should return response only for applicable agent types`() {
        // Given
        val metricId = UUID.randomUUID()
        val reportingMonth = YearMonth.now().minusMonths(1)

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

        val metricValue =
            InitiativeMetricValueEntity(
                initiativeMetricType = copilotMetricType,
                metricDirectory = metric,
                periodMonth = reportingMonth.atDay(1),
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
                reportingMonth = reportingMonth,
            )

        // Then
        assertEquals(1, result.size)

        val response = result.first()

        assertEquals(metricId, response.id)
        assertEquals("Метрика", response.name)
        assertEquals("шт", response.unit)
        assertEquals("growth", response.direction)
        assertEquals("copilot", response.agentType)
        assertEquals(true, response.isActive)
        assertEquals("Описание", response.description)
        assertEquals("monthly", response.frequency)
        assertEquals(BigDecimal.TEN, response.metricValue)
        assertEquals(BigDecimal.valueOf(20), response.targetValue)
        assertNull(response.planExecution)
        assertTrue(response.periods.isEmpty())
    }

    @Test
    fun `build should return separate responses for same metric with different agent types`() {
        // Given
        val metricId = UUID.randomUUID()
        val initiative = createInitiative(initiativeId = 1L)
        val reportingMonth = YearMonth.now().minusMonths(1)

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
                periodMonth = reportingMonth.atDay(1),
                metricValue = BigDecimal.valueOf(100),
                targetValue = BigDecimal.valueOf(200),
            ).apply {
                id = 100L
            }

        val copilotValue =
            InitiativeMetricValueEntity(
                initiativeMetricType = copilotMetricType,
                metricDirectory = metric,
                periodMonth = reportingMonth.atDay(1),
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
                metricValues = listOf(
                    autonomousValue,
                    copilotValue,
                ),
                reportingMonth = reportingMonth,
            )

        // Then
        assertEquals(2, result.size)

        val autonomousResponse =
            result.first { response ->
                response.agentType == "autonomous"
            }

        val copilotResponse =
            result.first { response ->
                response.agentType == "copilot"
            }

        assertEquals(
            BigDecimal.valueOf(100),
            autonomousResponse.metricValue,
        )
        assertEquals(
            BigDecimal.valueOf(200),
            autonomousResponse.targetValue,
        )
        assertNull(autonomousResponse.planExecution)
        assertTrue(autonomousResponse.periods.isEmpty())

        assertEquals(
            BigDecimal.valueOf(300),
            copilotResponse.metricValue,
        )
        assertEquals(
            BigDecimal.valueOf(400),
            copilotResponse.targetValue,
        )
        assertNull(copilotResponse.planExecution)
        assertTrue(copilotResponse.periods.isEmpty())
    }

    @Test
    fun `build should use reporting period value as main value and previous period as history`() {
        // Given
        val metricId = UUID.randomUUID()
        val initiative = createInitiative(initiativeId = 1L)

        val reportingMonth = YearMonth.now().minusMonths(1)
        val previousPeriodMonth = reportingMonth.minusMonths(1)

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

        val previousPeriodValue =
            InitiativeMetricValueEntity(
                initiativeMetricType = metricType,
                metricDirectory = metric,
                periodMonth = previousPeriodMonth.atDay(1),
                metricValue = BigDecimal.valueOf(10),
                targetValue = BigDecimal.valueOf(20),
            ).apply {
                id = 100L
            }

        val reportingPeriodValue =
            InitiativeMetricValueEntity(
                initiativeMetricType = metricType,
                metricDirectory = metric,
                periodMonth = reportingMonth.atDay(1),
                metricValue = BigDecimal.valueOf(30),
                targetValue = BigDecimal.valueOf(40),
            ).apply {
                id = 200L
            }

        // When
        val result =
            builder.build(
                metrics = listOf(metric),
                requestedAgentTypes =
                    setOf(InitiativeMetricAgentType.COPILOT),
                metricValues = listOf(
                    previousPeriodValue,
                    reportingPeriodValue,
                ),
                reportingMonth = reportingMonth,
            )

        // Then
        assertEquals(1, result.size)

        val response = result.first()

        assertEquals(
            BigDecimal.valueOf(30),
            response.metricValue,
        )
        assertEquals(
            BigDecimal.valueOf(40),
            response.targetValue,
        )
        assertNull(response.planExecution)

        assertEquals(1, response.periods.size)

        val previousPeriod = response.periods.single()

        assertEquals(
            previousPeriodMonth
                .atDay(1)
                .atStartOfDay(ZoneOffset.UTC)
                .toInstant(),
            previousPeriod.period,
        )

        assertEquals(
            BigDecimal.valueOf(10),
            previousPeriod.value,
        )
    }

    @Test
    fun `build should not use older period as history when previous period is absent`() {
        // Given
        val metricId = UUID.randomUUID()
        val initiative = createInitiative(initiativeId = 1L)

        val reportingMonth = YearMonth.now().minusMonths(1)
        val olderMonth = reportingMonth.minusMonths(2)

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

        val olderValue =
            InitiativeMetricValueEntity(
                initiativeMetricType = metricType,
                metricDirectory = metric,
                periodMonth = olderMonth.atDay(1),
                metricValue = BigDecimal.valueOf(10),
                targetValue = BigDecimal.valueOf(20),
            ).apply {
                id = 100L
            }

        val reportingPeriodValue =
            InitiativeMetricValueEntity(
                initiativeMetricType = metricType,
                metricDirectory = metric,
                periodMonth = reportingMonth.atDay(1),
                metricValue = BigDecimal.valueOf(30),
                targetValue = BigDecimal.valueOf(40),
            ).apply {
                id = 200L
            }

        // When
        val result =
            builder.build(
                metrics = listOf(metric),
                requestedAgentTypes =
                    setOf(InitiativeMetricAgentType.COPILOT),
                metricValues = listOf(
                    olderValue,
                    reportingPeriodValue,
                ),
                reportingMonth = reportingMonth,
            )

        // Then
        val response = result.single()

        assertEquals(
            BigDecimal.valueOf(30),
            response.metricValue,
        )
        assertEquals(
            BigDecimal.valueOf(40),
            response.targetValue,
        )

        assertTrue(response.periods.isEmpty())
    }

    @Test
    fun `build should return only previous reporting period in history`() {
        // Given
        val metricId = UUID.randomUUID()
        val initiative = createInitiative(initiativeId = 1L)

        val reportingMonth = YearMonth.now().minusMonths(1)
        val previousPeriodMonth = reportingMonth.minusMonths(1)
        val olderMonth = reportingMonth.minusMonths(2)

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

        val olderValue =
            InitiativeMetricValueEntity(
                initiativeMetricType = metricType,
                metricDirectory = metric,
                periodMonth = olderMonth.atDay(1),
                metricValue = BigDecimal.valueOf(5),
                targetValue = BigDecimal.valueOf(15),
            ).apply {
                id = 50L
            }

        val previousPeriodValue =
            InitiativeMetricValueEntity(
                initiativeMetricType = metricType,
                metricDirectory = metric,
                periodMonth = previousPeriodMonth.atDay(1),
                metricValue = BigDecimal.valueOf(10),
                targetValue = BigDecimal.valueOf(20),
            ).apply {
                id = 100L
            }

        val reportingPeriodValue =
            InitiativeMetricValueEntity(
                initiativeMetricType = metricType,
                metricDirectory = metric,
                periodMonth = reportingMonth.atDay(1),
                metricValue = BigDecimal.valueOf(30),
                targetValue = BigDecimal.valueOf(40),
            ).apply {
                id = 200L
            }

        // When
        val response =
            builder.build(
                metrics = listOf(metric),
                requestedAgentTypes =
                    setOf(InitiativeMetricAgentType.COPILOT),
                metricValues = listOf(
                    olderValue,
                    previousPeriodValue,
                    reportingPeriodValue,
                ),
                reportingMonth = reportingMonth,
            ).single()

        // Then
        assertEquals(1, response.periods.size)

        val period = response.periods.single()

        assertEquals(
            previousPeriodMonth
                .atDay(1)
                .atStartOfDay(ZoneOffset.UTC)
                .toInstant(),
            period.period,
        )

        assertEquals(
            BigDecimal.valueOf(10),
            period.value,
        )
    }

    @Test
    fun `build should calculate planExecute for increase metric`() {
        // Given
        val metricId = UUID.randomUUID()
        val initiative = createInitiative(initiativeId = 1L)
        val reportingMonth = YearMonth.now().minusMonths(1)

        val metric =
            createMetric(
                id = metricId,
                copilotApplicability = true,
                direction = "increase",
                frequency = "regular",
            )

        val metricType =
            InitiativeMetricTypeEntity(
                aiAgent = initiative,
                agentType = "copilot",
            ).apply {
                id = 10L
            }

        val metricValue =
            InitiativeMetricValueEntity(
                initiativeMetricType = metricType,
                metricDirectory = metric,
                periodMonth = reportingMonth.atDay(1),
                metricValue = BigDecimal.valueOf(30),
                targetValue = BigDecimal.valueOf(20),
            ).apply {
                id = 100L
            }

        // When
        val result =
            builder.build(
                metrics = listOf(metric),
                requestedAgentTypes =
                    setOf(InitiativeMetricAgentType.COPILOT),
                metricValues = listOf(metricValue),
                reportingMonth = reportingMonth,
            )

        // Then
        val response = result.first()

        assertEquals(true, response.planExecution)
        assertTrue(response.periods.isEmpty())
    }

    @Test
    fun `build should not return target value and planExecute for constant metric`() {
        // Given
        val metricId = UUID.randomUUID()
        val initiative = createInitiative(initiativeId = 1L)
        val reportingMonth = YearMonth.now().minusMonths(1)

        val metric =
            createMetric(
                id = metricId,
                copilotApplicability = true,
                direction = "increase",
                frequency = "constant",
            )

        val metricType =
            InitiativeMetricTypeEntity(
                aiAgent = initiative,
                agentType = "copilot",
            ).apply {
                id = 10L
            }

        val metricValue =
            InitiativeMetricValueEntity(
                initiativeMetricType = metricType,
                metricDirectory = metric,
                periodMonth = reportingMonth.atDay(1),
                metricValue = BigDecimal.valueOf(30),
                targetValue = BigDecimal.valueOf(20),
            ).apply {
                id = 100L
            }

        // When
        val result =
            builder.build(
                metrics = listOf(metric),
                requestedAgentTypes =
                    setOf(InitiativeMetricAgentType.COPILOT),
                metricValues = listOf(metricValue),
                reportingMonth = reportingMonth,
            )

        // Then
        val response = result.first()

        assertEquals(
            BigDecimal.valueOf(30),
            response.metricValue,
        )
        assertNull(response.targetValue)
        assertNull(response.planExecution)
        assertTrue(response.periods.isEmpty())
    }

    @Test
    fun `build should clear regular metric value when clearRegularMetricValue is true`() {
        // Given
        val metricId = UUID.randomUUID()
        val initiative = createInitiative(initiativeId = 1L)
        val reportingMonth = YearMonth.now().minusMonths(1)

        val metric =
            createMetric(
                id = metricId,
                copilotApplicability = true,
                frequency = "regular",
            )

        val metricType =
            InitiativeMetricTypeEntity(
                aiAgent = initiative,
                agentType = "copilot",
            ).apply {
                id = 10L
            }

        val metricValue =
            InitiativeMetricValueEntity(
                initiativeMetricType = metricType,
                metricDirectory = metric,
                periodMonth = reportingMonth.atDay(1),
                metricValue = BigDecimal.valueOf(30),
                targetValue = BigDecimal.valueOf(20),
            ).apply {
                id = 100L
            }

        // When
        val response =
            builder.build(
                metrics = listOf(metric),
                requestedAgentTypes =
                    setOf(InitiativeMetricAgentType.COPILOT),
                metricValues = listOf(metricValue),
                reportingMonth = reportingMonth,
                clearRegularMetricValue = true,
            ).single()

        // Then
        assertNull(response.metricValue)
        assertEquals(
            BigDecimal.valueOf(20),
            response.targetValue,
        )
        assertNull(response.planExecution)
    }

    @Test
    fun `buildWithoutValues should build responses with null metric values`() {
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
        assertEquals("autonomous", response.agentType)
        assertEquals(true, response.isActive)
        assertEquals("Описание", response.description)
        assertEquals("monthly", response.frequency)
        assertNull(response.metricValue)
        assertNull(response.targetValue)
        assertNull(response.planExecution)
        assertTrue(response.periods.isEmpty())
    }

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

    @Test
    fun `buildWithoutValues should build response for each applicable metric`() {
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
            )

        // When
        val result =
            builder.buildWithoutValues(
                metrics = listOf(
                    firstMetric,
                    secondMetric,
                ),
                requestedAgentTypes = requestedAgentTypes,
            )

        // Then
        assertEquals(1, result.size)

        val response = result.single()

        assertEquals("autonomous", response.agentType)
        assertEquals(firstMetric.id, response.id)
        assertNull(response.metricValue)
        assertNull(response.targetValue)
        assertNull(response.planExecution)
        assertTrue(response.periods.isEmpty())
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
        direction: String = "growth",
        frequency: String = "monthly",
    ): MetricsDirectoryEntity {
        return MetricsDirectoryEntity(
            name = "Метрика",
            unit = "шт",
            direction = direction,
            description = "Описание",
            frequency = frequency,
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
}
```
