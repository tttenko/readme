```java
@Test
fun `build should build periods with correct indexes by default`() {
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

    val currentMonth = YearMonth.now()
    val oldestHistoryMonth = currentMonth.minusMonths(2)
    val middleHistoryMonth = currentMonth.minusMonths(1)

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
            responseMonth = currentMonth,
        )

    // Then
    val response = result.first()
    val periods = response.periods

    assertEquals(BigDecimal.valueOf(30), response.metricValue)
    assertEquals(BigDecimal.valueOf(40), response.targetValue)

    assertEquals(3, periods.size)

    assertEquals(0, periods[0].index)
    assertEquals(BigDecimal.valueOf(30), periods[0].value)
    assertEquals(toUtcPeriod(currentMonth), periods[0].period)

    assertEquals(1, periods[1].index)
    assertEquals(BigDecimal.valueOf(10), periods[1].value)
    assertEquals(toUtcPeriod(oldestHistoryMonth), periods[1].period)

    assertEquals(2, periods[2].index)
    assertEquals(BigDecimal.valueOf(20), periods[2].value)
    assertEquals(toUtcPeriod(middleHistoryMonth), periods[2].period)
}
@Test
fun `build should exclude current period when includeCurrentPeriod is false`() {
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

    val currentMonth = YearMonth.now()
    val oldestHistoryMonth = currentMonth.minusMonths(2)
    val middleHistoryMonth = currentMonth.minusMonths(1)

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
            responseMonth = currentMonth,
            includeCurrentPeriod = false,
        )

    // Then
    val response = result.first()
    val periods = response.periods

    assertEquals(BigDecimal.valueOf(30), response.metricValue)
    assertEquals(BigDecimal.valueOf(40), response.targetValue)

    assertEquals(2, periods.size)

    assertEquals(1, periods[0].index)
    assertEquals(BigDecimal.valueOf(10), periods[0].value)
    assertEquals(toUtcPeriod(oldestHistoryMonth), periods[0].period)

    assertEquals(2, periods[1].index)
    assertEquals(BigDecimal.valueOf(20), periods[1].value)
    assertEquals(toUtcPeriod(middleHistoryMonth), periods[1].period)
}
```
