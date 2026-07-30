```java
@ExtendWith(MockKExtension::class)
internal class StageDeadlineExpiredStrategyTest {

    private val strategy = StageDeadlineExpiredStrategy()

    @Test
    fun `should return only initiative with expired unfinished stage`() {
        val today = LocalDate.of(2026, 7, 30)
        val session = mockk<InitiativeDeviationCalculationSession>()

        every { session.statusSlaByInitiativeId } returns mapOf(
            1L to listOf(
                createStatusSla(
                    plannedDate = today.minusDays(1).atStartOfDay(),
                    completedDate = null,
                )
            ),
            2L to listOf(
                createStatusSla(
                    plannedDate = today.minusDays(1).atStartOfDay(),
                    completedDate = today.atStartOfDay(),
                )
            ),
            3L to listOf(
                createStatusSla(
                    plannedDate = today.atStartOfDay(),
                    completedDate = null,
                )
            ),
        )

        val result = strategy.findMatchingInitiativeIds(
            candidateInitiativeIds = setOf(1L, 2L, 3L, 4L),
            session = session,
            context = createContext(today),
        )

        assertThat(result).containsExactly(1L)
        assertThat(strategy.code)
            .isEqualTo(InitiativeDeviationCode.STAGE_DEADLINE_EXPIRED)
        assertThat(strategy.evaluationOrder).isEqualTo(10)
    }
}
Не заполнены дедлайны
@ExtendWith(MockKExtension::class)
internal class StageDeadlinesNotFilledStrategyTest {

    private val strategy = StageDeadlinesNotFilledStrategy()

    @Test
    fun `should return initiative when unfinished stage has no planned date`() {
        val today = LocalDate.of(2026, 7, 30)
        val session = mockk<InitiativeDeviationCalculationSession>()

        every { session.statusSlaByInitiativeId } returns mapOf(
            1L to listOf(
                createStatusSla(
                    plannedDate = null,
                    completedDate = null,
                )
            ),
            2L to listOf(
                createStatusSla(
                    plannedDate = null,
                    completedDate = today.atStartOfDay(),
                )
            ),
            3L to listOf(
                createStatusSla(
                    plannedDate = today.plusDays(5).atStartOfDay(),
                    completedDate = null,
                )
            ),
        )

        val result = strategy.findMatchingInitiativeIds(
            candidateInitiativeIds = setOf(1L, 2L, 3L, 4L),
            session = session,
            context = createContext(today),
        )

        assertThat(result).containsExactly(1L)
        assertThat(strategy.code)
            .isEqualTo(InitiativeDeviationCode.STAGE_DEADLINES_NOT_FILLED)
        assertThat(strategy.evaluationOrder).isEqualTo(20)
    }
}
Дедлайн через один, два или три дня

Одним параметризованным тестом покрываются все три реализации AbstractUpcomingStageDeadlineStrategy.

@ExtendWith(MockKExtension::class)
internal class UpcomingStageDeadlineStrategyTest {

    @ParameterizedTest
    @MethodSource("strategies")
    fun `should return initiative with unfinished stage on expected date`(
        strategy: InitiativeDeviationStrategy,
        daysBeforeDeadline: Long,
        expectedCode: InitiativeDeviationCode,
        expectedOrder: Int,
    ) {
        val today = LocalDate.of(2026, 7, 30)
        val expectedDate = today.plusDays(daysBeforeDeadline)
        val session = mockk<InitiativeDeviationCalculationSession>()

        every { session.statusSlaByInitiativeId } returns mapOf(
            1L to listOf(
                createStatusSla(
                    plannedDate = expectedDate.atStartOfDay(),
                    completedDate = null,
                )
            ),
            2L to listOf(
                createStatusSla(
                    plannedDate = expectedDate.atStartOfDay(),
                    completedDate = today.atStartOfDay(),
                )
            ),
            3L to listOf(
                createStatusSla(
                    plannedDate = expectedDate.plusDays(1).atStartOfDay(),
                    completedDate = null,
                )
            ),
        )

        val result = strategy.findMatchingInitiativeIds(
            candidateInitiativeIds = setOf(1L, 2L, 3L, 4L),
            session = session,
            context = createContext(today),
        )

        assertThat(result).containsExactly(1L)
        assertThat(strategy.code).isEqualTo(expectedCode)
        assertThat(strategy.evaluationOrder).isEqualTo(expectedOrder)
    }

    companion object {

        @JvmStatic
        fun strategies(): Stream<Arguments> =
            Stream.of(
                Arguments.of(
                    StageDeadlineTomorrowStrategy(),
                    1L,
                    InitiativeDeviationCode.STAGE_DEADLINE_TOMORROW,
                    50,
                ),
                Arguments.of(
                    StageDeadlineInTwoDaysStrategy(),
                    2L,
                    InitiativeDeviationCode.STAGE_DEADLINE_IN_2_DAYS,
                    60,
                ),
                Arguments.of(
                    StageDeadlineInThreeDaysStrategy(),
                    3L,
                    InitiativeDeviationCode.STAGE_DEADLINE_IN_3_DAYS,
                    70,
                ),
            )
    }
}
Энейблеры
@ExtendWith(MockKExtension::class)
internal class EnablersNotFilledStrategyTest {

    private val strategy = EnablersNotFilledStrategy()

    @Test
    fun `should return initiatives without enablers`() {
        val session = mockk<InitiativeDeviationCalculationSession>()

        every {
            session.initiativeIdsWithEnablers
        } returns setOf(2L, 4L)

        val result = strategy.findMatchingInitiativeIds(
            candidateInitiativeIds = setOf(1L, 2L, 3L, 4L),
            session = session,
            context = createContext(),
        )

        assertThat(result).containsExactlyInAnyOrder(1L, 3L)
        assertThat(strategy.code)
            .isEqualTo(InitiativeDeviationCode.ENABLERS_NOT_FILLED)
        assertThat(strategy.evaluationOrder).isEqualTo(40)
    }
}
GigaUsage
@ExtendWith(MockKExtension::class)
internal class GigaUsageNotFilledStrategyTest {

    private val strategy = GigaUsageNotFilledStrategy()

    @Test
    fun `should return initiatives without valid gigausage`() {
        val session = mockk<InitiativeDeviationCalculationSession>()

        every {
            session.initiativeIdsWithValidGigaUsage
        } returns setOf(1L, 3L)

        val result = strategy.findMatchingInitiativeIds(
            candidateInitiativeIds = setOf(1L, 2L, 3L, 4L),
            session = session,
            context = createContext(),
        )

        assertThat(result).containsExactlyInAnyOrder(2L, 4L)
        assertThat(strategy.code)
            .isEqualTo(InitiativeDeviationCode.GIGAUSAGE_NOT_FILLED)
        assertThat(strategy.evaluationOrder).isEqualTo(30)
    }
}
Метрики до порогового дня
@ExtendWith(MockKExtension::class)
internal class CurrentPeriodMetricsNotFilledStrategyTest {

    private val strategy = CurrentPeriodMetricsNotFilledStrategy()

    @Test
    fun `should not check metrics on or before configured deadline day`() {
        val finder = mockk<CurrentPeriodMetricsDeviationFinder>()
        val session = mockk<InitiativeDeviationCalculationSession>()

        val context = createContext(
            today = LocalDate.of(2026, 7, 15),
            metricsDeadlineDay = 15,
        )

        val result = strategy.findMatchingInitiativeIds(
            candidateInitiativeIds = setOf(1L, 2L),
            session = session,
            context = context,
        )

        assertThat(result).isEmpty()

        verify(exactly = 0) {
            finder.findInitiativeIdsWithMissingMetrics(
                initiativeIds = any(),
                currentPeriod = any(),
            )
        }
    }

    @Test
    fun `should check metrics only for targetSolution initiatives after deadline day`() {
        val finder = mockk<CurrentPeriodMetricsDeviationFinder>()
        val session = mockk<InitiativeDeviationCalculationSession>()

        val today = LocalDate.of(2026, 7, 16)
        val currentPeriod = LocalDate.of(2026, 7, 1)

        every {
            session.statusCodeByInitiativeId
        } returns mapOf(
            1L to TARGET_SOLUTION,
            2L to "implementation",
            3L to TARGET_SOLUTION,
        )

        every {
            session.currentPeriodMetricsDeviationFinder
        } returns finder

        every {
            finder.findInitiativeIdsWithMissingMetrics(
                initiativeIds = setOf(1L, 3L),
                currentPeriod = currentPeriod,
            )
        } returns setOf(3L)

        val result = strategy.findMatchingInitiativeIds(
            candidateInitiativeIds = setOf(1L, 2L, 3L),
            session = session,
            context = InitiativeDeviationEvaluationContext(
                today = today,
                currentPeriod = currentPeriod,
                metricsDeadlineDay = 15,
            ),
        )

        assertThat(result).containsExactly(3L)
        assertThat(strategy.code)
            .isEqualTo(
                InitiativeDeviationCode.CURRENT_PERIOD_METRICS_NOT_FILLED
            )
        assertThat(strategy.evaluationOrder).isEqualTo(1000)

        verify(exactly = 1) {
            finder.findInitiativeIdsWithMissingMetrics(
                initiativeIds = setOf(1L, 3L),
                currentPeriod = currentPeriod,
            )
        }
    }
}

В первом тесте метрик мок finder не связан с session, поэтому проверка его отсутствия будет формально бессмысленной. Корректнее связать его:

@Test
fun `should not check metrics on or before configured deadline day`() {
    val finder = mockk<CurrentPeriodMetricsDeviationFinder>()
    val session = mockk<InitiativeDeviationCalculationSession>()

    every {
        session.currentPeriodMetricsDeviationFinder
    } returns finder

    val result = strategy.findMatchingInitiativeIds(
        candidateInitiativeIds = setOf(1L, 2L),
        session = session,
        context = createContext(
            today = LocalDate.of(2026, 7, 15),
            metricsDeadlineDay = 15,
        ),
    )

    assertThat(result).isEmpty()

    verify(exactly = 0) {
        finder.findInitiativeIdsWithMissingMetrics(
            initiativeIds = any(),
            currentPeriod = any(),
        )
    }
}

Используй именно эту исправленную версию первого теста.

Реестр стратегий

Один тест одновременно проверяет:

исключение выключенной стратегии;
сортировку включённых стратегий по evaluationOrder.
@ExtendWith(MockKExtension::class)
internal class InitiativeDeviationStrategyRegistryTest {

    @Test
    fun `should return only enabled strategies sorted by evaluation order`() {
        val properties = mockk<InitiativeDeviationProperties>()

        val firstStrategy = mockStrategy(
            code = InitiativeDeviationCode.STAGE_DEADLINE_EXPIRED,
            evaluationOrder = 10,
        )

        val disabledStrategy = mockStrategy(
            code = InitiativeDeviationCode.STAGE_DEADLINES_NOT_FILLED,
            evaluationOrder = 20,
        )

        val lastStrategy = mockStrategy(
            code =
                InitiativeDeviationCode.CURRENT_PERIOD_METRICS_NOT_FILLED,
            evaluationOrder = 1000,
        )

        every {
            properties.getRequiredRule(
                InitiativeDeviationCode.STAGE_DEADLINE_EXPIRED
            ).enabled
        } returns true

        every {
            properties.getRequiredRule(
                InitiativeDeviationCode.STAGE_DEADLINES_NOT_FILLED
            ).enabled
        } returns false

        every {
            properties.getRequiredRule(
                InitiativeDeviationCode.CURRENT_PERIOD_METRICS_NOT_FILLED
            ).enabled
        } returns true

        val registry = InitiativeDeviationStrategyRegistry(
            strategies = listOf(
                lastStrategy,
                disabledStrategy,
                firstStrategy,
            ),
            initiativeDeviationProperties = properties,
        )

        val result = registry.getEnabledStrategies()

        assertThat(result)
            .containsExactly(firstStrategy, lastStrategy)
    }

    private fun mockStrategy(
        code: InitiativeDeviationCode,
        evaluationOrder: Int,
    ): InitiativeDeviationStrategy {
        val strategy = mockk<InitiativeDeviationStrategy>()

        every { strategy.code } returns code
        every { strategy.evaluationOrder } returns evaluationOrder

        return strategy
    }
}

Если MockK не позволяет стабильно замокать цепочку:

properties.getRequiredRule(code).enabled

создай отдельные моки правил:

val enabledRule =
    mockk<InitiativeDeviationProperties.Rule>()

val disabledRule =
    mockk<InitiativeDeviationProperties.Rule>()

every { enabledRule.enabled } returns true
every { disabledRule.enabled } returns false

every {
    properties.getRequiredRule(
        InitiativeDeviationCode.STAGE_DEADLINE_EXPIRED
    )
} returns enabledRule

every {
    properties.getRequiredRule(
        InitiativeDeviationCode.STAGE_DEADLINES_NOT_FILLED
    )
} returns disabledRule

every {
    properties.getRequiredRule(
        InitiativeDeviationCode.CURRENT_PERIOD_METRICS_NOT_FILLED
    )
} returns enabledRule
Общие вспомогательные функции
private fun createContext(
    today: LocalDate = LocalDate.of(2026, 7, 30),
    metricsDeadlineDay: Int = 15,
): InitiativeDeviationEvaluationContext =
    InitiativeDeviationEvaluationContext(
        today = today,
        currentPeriod = YearMonth.from(today).atDay(1),
        metricsDeadlineDay = metricsDeadlineDay,
    )

private fun createStatusSla(
    plannedDate: LocalDateTime?,
    completedDate: LocalDateTime?,
): AgentStatusSlaEntity =
    AgentStatusSlaEntity().apply {
        this.plannedDate = plannedDate
        this.completedDate = completedDate
    }

    InitiativeDeviationStrategyTest.kt
```
