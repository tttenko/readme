```java

interface InitiativeDeviationStrategy {

    /**
     * Код отклонения, за которое отвечает стратегия.
     */
    val code: InitiativeDeviationCode

    /**
     * Порядок технической проверки.
     *
     * Не совпадает с приоритетом показа.
     * Тяжёлые проверки должны выполняться последними.
     */
    val evaluationOrder: Int

    /**
     * Возвращает ID инициатив, для которых найдено отклонение.
     */
    fun findMatchingInitiativeIds(
        candidateInitiativeIds: Set<Long>,
        session: InitiativeDeviationCalculationSession,
        context: InitiativeDeviationEvaluationContext,
    ): Set<Long>
}

```
