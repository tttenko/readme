```java
@Test
fun `should return only initiative with expired unfinished current or future stage`() {
    val today = LocalDate.of(2026, 7, 30)
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
        // Просроченный незавершённый текущий этап — должен попасть
        1L to listOf(
            createStatusSla(
                statusId = 20L,
                plannedDate = today.minusDays(1).atStartOfDay(),
                completedDate = null,
            )
        ),

        // Просроченный, но завершённый этап — не должен попасть
        2L to listOf(
            createStatusSla(
                statusId = 20L,
                plannedDate = today.minusDays(1).atStartOfDay(),
                completedDate = today.atStartOfDay(),
            )
        ),

        // Дедлайн сегодня — ещё не просрочен
        3L to listOf(
            createStatusSla(
                statusId = 20L,
                plannedDate = today.atStartOfDay(),
                completedDate = null,
            )
        ),

        // Просроченный прошедший этап — не должен учитываться
        5L to listOf(
            createStatusSla(
                statusId = 10L,
                plannedDate = today.minusDays(1).atStartOfDay(),
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
    assertThat(strategy.code)
        .isEqualTo(InitiativeDeviationCode.STAGE_DEADLINE_EXPIRED)
    assertThat(strategy.evaluationOrder).isEqualTo(10)
}
  ```
