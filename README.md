```java
internal class StageDeadlinesNotFilledStrategyTest {

    private val strategy = StageDeadlinesNotFilledStrategy()

    private val today = LocalDate.of(2026, 7, 30)
    private val context = createContext(today)

    @Test
    fun `should return initiative when unfinished stage has no planned date`() {
        val session = mockk<InitiativeDeviationCalculationSession>()

        every { session.expectedStageIdsByInitiativeId } returns mapOf(
            1L to setOf(30L),
            2L to setOf(30L),
            3L to setOf(30L),
        )

        every { session.statusSlaByInitiativeId } returns mapOf(
            1L to listOf(
                createStatusSla(
                    statusId = 30L,
                    plannedDate = null,
                    completedDate = null,
                )
            ),
            2L to listOf(
                createStatusSla(
                    statusId = 30L,
                    plannedDate = null,
                    completedDate = today.atStartOfDay(),
                )
            ),
            3L to listOf(
                createStatusSla(
                    statusId = 30L,
                    plannedDate = today.plusDays(5).atStartOfDay(),
                    completedDate = null,
                )
            ),
        )

        val result = strategy.findMatchingInitiativeIds(
            candidateInitiativeIds = setOf(1L, 2L, 3L),
            session = session,
            context = context,
        )

        assertThat(result).containsExactly(1L)
        assertThat(strategy.code)
            .isEqualTo(InitiativeDeviationCode.STAGE_DEADLINES_NOT_FILLED)
        assertThat(strategy.evaluationOrder).isEqualTo(20)
    }

    @Test
    fun `should return initiative when SLA record for expected stage is missing`() {
        val session = mockk<InitiativeDeviationCalculationSession>()

        every { session.expectedStageIdsByInitiativeId } returns mapOf(
            1L to setOf(30L, 40L),
        )

        every { session.statusSlaByInitiativeId } returns mapOf(
            1L to listOf(
                createStatusSla(
                    statusId = 30L,
                    plannedDate = today.plusDays(1).atStartOfDay(),
                    completedDate = null,
                )
            )
        )

        val result = strategy.findMatchingInitiativeIds(
            candidateInitiativeIds = setOf(1L),
            session = session,
            context = context,
        )

        assertThat(result).containsExactly(1L)
    }

    @Test
    fun `should not return initiative when all expected stages have planned dates`() {
        val session = mockk<InitiativeDeviationCalculationSession>()

        every { session.expectedStageIdsByInitiativeId } returns mapOf(
            1L to setOf(30L, 40L, 50L),
        )

        every { session.statusSlaByInitiativeId } returns mapOf(
            1L to listOf(
                createStatusSla(
                    statusId = 30L,
                    plannedDate = today.plusDays(1).atStartOfDay(),
                    completedDate = null,
                ),
                createStatusSla(
                    statusId = 40L,
                    plannedDate = today.plusDays(2).atStartOfDay(),
                    completedDate = null,
                ),
                createStatusSla(
                    statusId = 50L,
                    plannedDate = today.plusDays(3).atStartOfDay(),
                    completedDate = null,
                ),
            )
        )

        val result = strategy.findMatchingInitiativeIds(
            candidateInitiativeIds = setOf(1L),
            session = session,
            context = context,
        )

        assertThat(result).isEmpty()
    }

    @Test
    fun `should not return initiative when stage without planned date is completed`() {
        val session = mockk<InitiativeDeviationCalculationSession>()

        every { session.expectedStageIdsByInitiativeId } returns mapOf(
            1L to setOf(30L),
        )

        every { session.statusSlaByInitiativeId } returns mapOf(
            1L to listOf(
                createStatusSla(
                    statusId = 30L,
                    plannedDate = null,
                    completedDate = today.atStartOfDay(),
                )
            )
        )

        val result = strategy.findMatchingInitiativeIds(
            candidateInitiativeIds = setOf(1L),
            session = session,
            context = context,
        )

        assertThat(result).isEmpty()
    }

    @Test
    fun `should ignore SLA without planned date for past stage`() {
        val session = mockk<InitiativeDeviationCalculationSession>()

        every { session.expectedStageIdsByInitiativeId } returns mapOf(
            1L to setOf(40L),
        )

        every { session.statusSlaByInitiativeId } returns mapOf(
            1L to listOf(
                createStatusSla(
                    statusId = 30L,
                    plannedDate = null,
                    completedDate = null,
                ),
                createStatusSla(
                    statusId = 40L,
                    plannedDate = today.plusDays(5).atStartOfDay(),
                    completedDate = null,
                ),
            )
        )

        val result = strategy.findMatchingInitiativeIds(
            candidateInitiativeIds = setOf(1L),
            session = session,
            context = context,
        )

        assertThat(result).isEmpty()
    }

    @Test
    fun `should not return initiative when expected stages are absent`() {
        val session = mockk<InitiativeDeviationCalculationSession>()

        every { session.expectedStageIdsByInitiativeId } returns emptyMap()
        every { session.statusSlaByInitiativeId } returns emptyMap()

        val result = strategy.findMatchingInitiativeIds(
            candidateInitiativeIds = setOf(1L),
            session = session,
            context = context,
        )

        assertThat(result).isEmpty()
    }

    private fun createStatusSla(
        statusId: Long,
        plannedDate: LocalDateTime?,
        completedDate: LocalDateTime?,
    ): AgentStatusSlaEntity =
        AgentStatusSlaEntity().apply {
            primaryKey.agentStatusId = statusId
            this.plannedDate = plannedDate
            this.completedDate = completedDate
        }
}
  ```
