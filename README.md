```java
@ParameterizedTest
@MethodSource("strategies")
fun `should return initiative with unfinished current or future stage on expected date`(
    strategy: InitiativeDeviationStrategy,
    daysBeforeDeadline: Long,
    expectedCode: InitiativeDeviationCode,
    expectedOrder: Int,
) {
    val today = LocalDate.of(2026, 7, 30)
    val expectedDate = today.plusDays(daysBeforeDeadline)
    val session = mockk<InitiativeDeviationCalculationSession>()

    every { session.statusOrderingByInitiativeId } returns mapOf(
        1L to 30L,
        2L to 30L,
        3L to 30L,
        4L to 30L,
        5L to 30L,
    )

    every { session.activeStatusOrderingById } returns mapOf(
        10L to 20L, // прошедший этап
        20L to 30L, // текущий этап
        30L to 40L, // будущий этап
    )

    every { session.statusSlaByInitiativeId } returns mapOf(
        1L to listOf(
            createStatusSla(
                statusId = 20L,
                plannedDate = expectedDate.atStartOfDay(),
                completedDate = null,
            )
        ),
        2L to listOf(
            createStatusSla(
                statusId = 20L,
                plannedDate = expectedDate.atStartOfDay(),
                completedDate = today.atStartOfDay(),
            )
        ),
        3L to listOf(
            createStatusSla(
                statusId = 30L,
                plannedDate = expectedDate.plusDays(1).atStartOfDay(),
                completedDate = null,
            )
        ),
        5L to listOf(
            createStatusSla(
                statusId = 10L,
                plannedDate = expectedDate.atStartOfDay(),
                completedDate = null,
            )
        ),
    )

    val result = strategy.findMatchingInitiativeIds(
        candidateInitiativeIds = setOf(1L, 2L, 3L, 4L, 5L),
        session = session,
        context = createContext(today),
    )

    assertThat(result).containsExactly(1L)
    assertThat(strategy.code).isEqualTo(expectedCode)
    assertThat(strategy.evaluationOrder).isEqualTo(expectedOrder)
}
  ```
