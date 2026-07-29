```java

@Component
class CurrentPeriodMetricsNotFilledStrategy : InitiativeDeviationStrategy {

    override val code =
        InitiativeDeviationCode.CURRENT_PERIOD_METRICS_NOT_FILLED

    /**
     * Самая тяжёлая проверка выполняется последней.
     */
    override val evaluationOrder: Int = 1000

    override fun findMatchingInitiativeIds(
        candidateInitiativeIds: Set<Long>,
        session: InitiativeDeviationCalculationSession,
        context: InitiativeDeviationEvaluationContext,
    ): Set<Long> {
        if (context.today.dayOfMonth <= context.metricsDeadlineDay) {
            return emptySet()
        }

        val targetSolutionInitiativeIds =
            candidateInitiativeIds.filterTo(mutableSetOf()) { initiativeId ->
                session.statusCodeByInitiativeId[initiativeId] ==
                    Metadata.DraftStatus.TARGET_SOLUTION
            }

        return session.currentPeriodMetricsDeviationFinder
            .findInitiativeIdsWithMissingMetrics(
                initiativeIds = targetSolutionInitiativeIds,
                currentPeriod = context.currentPeriod,
            )
    }
}

@Component
class InitiativeDeviationStrategyRegistry(
    strategies: List<InitiativeDeviationStrategy>,
) {

    private val strategiesByCode =
        strategies.associateBy { strategy -> strategy.code }

    init {
        require(strategiesByCode.size == strategies.size) {
            "Обнаружено несколько стратегий для одного кода отклонения"
        }
    }

    fun getEnabledStrategies(
        properties: InitiativeDeviationProperties,
    ): List<InitiativeDeviationStrategy> =
        strategiesByCode.values
            .filter { strategy ->
                properties.getRequiredRule(strategy.code).enabled
            }
            .sortedBy { strategy -> strategy.evaluationOrder }
}

@Component
class InitiativeDeviationCalculator(
    private val strategyRegistry: InitiativeDeviationStrategyRegistry,
    private val sessionFactory: InitiativeDeviationCalculationSessionFactory,
    private val properties: InitiativeDeviationProperties,
    private val clock: Clock,
) {

    /**
     * Рассчитывает полный список отклонений для переданных инициатив.
     *
     * Используется при формировании content страницы.
     */
    fun calculate(
        initiativeIds: Set<Long>,
    ): Map<Long, List<InitiativeDeviationResponse>> {
        if (initiativeIds.isEmpty()) {
            return emptyMap()
        }

        val context = createContext()
        val session = sessionFactory.create(initiativeIds)

        val deviationsByInitiativeId =
            initiativeIds.associateWith {
                mutableListOf<InitiativeDeviationResponse>()
            }

        strategyRegistry
            .getEnabledStrategies(properties)
            .forEach { strategy ->
                val matchingIds =
                    strategy.findMatchingInitiativeIds(
                        candidateInitiativeIds = initiativeIds,
                        session = session,
                        context = context,
                    )

                val rule = properties.getRequiredRule(strategy.code)

                val response =
                    InitiativeDeviationResponse(
                        code = strategy.code.name,
                        priority = rule.priority,
                        title = rule.title,
                        description = rule.description,
                    )

                matchingIds.forEach { initiativeId ->
                    deviationsByInitiativeId
                        .getValue(initiativeId)
                        .add(response)
                }
            }

        return deviationsByInitiativeId.mapValues { (_, deviations) ->
            deviations.sortedBy { deviation -> deviation.priority }
        }
    }

    /**
     * Определяет инициативы, у которых есть хотя бы одно отклонение.
     *
     * Используется для hasDeviation=true до применения пагинации.
     * После первого найденного отклонения инициатива исключается
     * из последующих проверок.
     */
    fun findInitiativeIdsWithAnyDeviation(
        initiativeIds: Set<Long>,
    ): Set<Long> {
        if (initiativeIds.isEmpty()) {
            return emptySet()
        }

        val context = createContext()
        val session = sessionFactory.create(initiativeIds)

        val matchedIds = mutableSetOf<Long>()
        var remainingIds = initiativeIds

        strategyRegistry
            .getEnabledStrategies(properties)
            .forEach { strategy ->
                if (remainingIds.isEmpty()) {
                    return@forEach
                }

                val currentMatches =
                    strategy.findMatchingInitiativeIds(
                        candidateInitiativeIds = remainingIds,
                        session = session,
                        context = context,
                    )

                matchedIds += currentMatches
                remainingIds -= currentMatches
            }

        return matchedIds
    }

    private fun createContext(): InitiativeDeviationEvaluationContext {
        val today = LocalDate.now(clock)

        return InitiativeDeviationEvaluationContext(
            today = today,
            currentPeriod = YearMonth.from(today).atDay(1),
            metricsDeadlineDay =
                properties.currentPeriodMetricsDeadlineDay,
        )
    }
}




```
