```java
internal class StageDeadlinesNotFilledStrategyTest {

    private val strategy = StageDeadlinesNotFilledStrategy()

    private val today = LocalDate.of(2026, 7, 30)
    private val context = mockk<InitiativeDeviationEvaluationContext>(relaxed = true)

    @Test
    fun `should return initiative when unfinished current stage has no planned date`() {
        val session = mockk<InitiativeDeviationCalculationSession>()

        every { session.statusOrderingByInitiativeId } returns mapOf(
            1L to 30L,
        )
        every { session.activeStatuses } returns listOf(
            createStatus(id = 3L, ordering = 30L),
        )
        every { session.statusSlaByInitiativeId } returns mapOf(
            1L to listOf(
                createStatusSla(
                    initiativeId = 1L,
                    statusId = 3L,
                    plannedDate = null,
                    completedDate = null,
                ),
            ),
        )

        val result = strategy.findMatchingInitiativeIds(
            candidateInitiativeIds = setOf(1L),
            session = session,
            context = context,
        )

        assertThat(result).containsExactly(1L)
        assertThat(strategy.code)
            .isEqualTo(InitiativeDeviationCode.STAGE_DEADLINES_NOT_FILLED)
        assertThat(strategy.evaluationOrder).isEqualTo(20)
    }

    @Test
    fun `should return initiative when SLA record for current stage is missing`() {
        val session = mockk<InitiativeDeviationCalculationSession>()

        every { session.statusOrderingByInitiativeId } returns mapOf(
            1L to 30L,
        )
        every { session.activeStatuses } returns listOf(
            createStatus(id = 3L, ordering = 30L),
        )
        every { session.statusSlaByInitiativeId } returns emptyMap()

        val result = strategy.findMatchingInitiativeIds(
            candidateInitiativeIds = setOf(1L),
            session = session,
            context = context,
        )

        assertThat(result).containsExactly(1L)
    }

    @Test
    fun `should return initiative when SLA record for future stage is missing`() {
        val session = mockk<InitiativeDeviationCalculationSession>()

        every { session.statusOrderingByInitiativeId } returns mapOf(
            1L to 30L,
        )
        every { session.activeStatuses } returns listOf(
            createStatus(id = 3L, ordering = 30L),
            createStatus(id = 4L, ordering = 40L),
        )
        every { session.statusSlaByInitiativeId } returns mapOf(
            1L to listOf(
                createStatusSla(
                    initiativeId = 1L,
                    statusId = 3L,
                    plannedDate = today.plusDays(5).atStartOfDay(),
                    completedDate = null,
                ),
            ),
        )

        val result = strategy.findMatchingInitiativeIds(
            candidateInitiativeIds = setOf(1L),
            session = session,
            context = context,
        )

        assertThat(result).containsExactly(1L)
    }

    @Test
    fun `should not return initiative when all current and future stages have planned dates`() {
        val session = mockk<InitiativeDeviationCalculationSession>()

        every { session.statusOrderingByInitiativeId } returns mapOf(
            1L to 30L,
        )
        every { session.activeStatuses } returns listOf(
            createStatus(id = 3L, ordering = 30L),
            createStatus(id = 4L, ordering = 40L),
            createStatus(id = 5L, ordering = 50L),
        )
        every { session.statusSlaByInitiativeId } returns mapOf(
            1L to listOf(
                createStatusSla(
                    initiativeId = 1L,
                    statusId = 3L,
                    plannedDate = today.plusDays(5).atStartOfDay(),
                    completedDate = null,
                ),
                createStatusSla(
                    initiativeId = 1L,
                    statusId = 4L,
                    plannedDate = today.plusDays(10).atStartOfDay(),
                    completedDate = null,
                ),
                createStatusSla(
                    initiativeId = 1L,
                    statusId = 5L,
                    plannedDate = today.plusDays(15).atStartOfDay(),
                    completedDate = null,
                ),
            ),
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

        every { session.statusOrderingByInitiativeId } returns mapOf(
            1L to 30L,
        )
        every { session.activeStatuses } returns listOf(
            createStatus(id = 3L, ordering = 30L),
        )
        every { session.statusSlaByInitiativeId } returns mapOf(
            1L to listOf(
                createStatusSla(
                    initiativeId = 1L,
                    statusId = 3L,
                    plannedDate = null,
                    completedDate = today.atStartOfDay(),
                ),
            ),
        )

        val result = strategy.findMatchingInitiativeIds(
            candidateInitiativeIds = setOf(1L),
            session = session,
            context = context,
        )

        assertThat(result).isEmpty()
    }

    @Test
    fun `should ignore unfinished stage before current status`() {
        val session = mockk<InitiativeDeviationCalculationSession>()

        every { session.statusOrderingByInitiativeId } returns mapOf(
            1L to 40L,
        )
        every { session.activeStatuses } returns listOf(
            createStatus(id = 3L, ordering = 30L),
            createStatus(id = 4L, ordering = 40L),
        )
        every { session.statusSlaByInitiativeId } returns mapOf(
            1L to listOf(
                createStatusSla(
                    initiativeId = 1L,
                    statusId = 3L,
                    plannedDate = null,
                    completedDate = null,
                ),
                createStatusSla(
                    initiativeId = 1L,
                    statusId = 4L,
                    plannedDate = today.plusDays(5).atStartOfDay(),
                    completedDate = null,
                ),
            ),
        )

        val result = strategy.findMatchingInitiativeIds(
            candidateInitiativeIds = setOf(1L),
            session = session,
            context = context,
        )

        assertThat(result).isEmpty()
    }

    @Test
    fun `should ignore initiative without current status ordering`() {
        val session = mockk<InitiativeDeviationCalculationSession>()

        every { session.statusOrderingByInitiativeId } returns emptyMap()

        val result = strategy.findMatchingInitiativeIds(
            candidateInitiativeIds = setOf(1L),
            session = session,
            context = context,
        )

        assertThat(result).isEmpty()

        verify(exactly = 0) {
            session.activeStatuses
            session.statusSlaByInitiativeId
        }
    }

    @Test
    fun `should ignore active status without ordering`() {
        val session = mockk<InitiativeDeviationCalculationSession>()

        every { session.statusOrderingByInitiativeId } returns mapOf(
            1L to 30L,
        )
        every { session.activeStatuses } returns listOf(
            createStatus(id = 3L, ordering = null),
        )
        every { session.statusSlaByInitiativeId } returns emptyMap()

        val result = strategy.findMatchingInitiativeIds(
            candidateInitiativeIds = setOf(1L),
            session = session,
            context = context,
        )

        assertThat(result).isEmpty()
    }

    @Test
    fun `should calculate deviations independently for several initiatives`() {
        val session = mockk<InitiativeDeviationCalculationSession>()

        every { session.statusOrderingByInitiativeId } returns mapOf(
            1L to 30L,
            2L to 40L,
            3L to 50L,
        )
        every { session.activeStatuses } returns listOf(
            createStatus(id = 3L, ordering = 30L),
            createStatus(id = 4L, ordering = 40L),
            createStatus(id = 5L, ordering = 50L),
        )
        every { session.statusSlaByInitiativeId } returns mapOf(
            1L to listOf(
                createStatusSla(
                    initiativeId = 1L,
                    statusId = 3L,
                    plannedDate = null,
                    completedDate = null,
                ),
                createStatusSla(
                    initiativeId = 1L,
                    statusId = 4L,
                    plannedDate = today.plusDays(5).atStartOfDay(),
                    completedDate = null,
                ),
                createStatusSla(
                    initiativeId = 1L,
                    statusId = 5L,
                    plannedDate = today.plusDays(10).atStartOfDay(),
                    completedDate = null,
                ),
            ),
            2L to listOf(
                createStatusSla(
                    initiativeId = 2L,
                    statusId = 4L,
                    plannedDate = today.plusDays(5).atStartOfDay(),
                    completedDate = null,
                ),
                createStatusSla(
                    initiativeId = 2L,
                    statusId = 5L,
                    plannedDate = today.plusDays(10).atStartOfDay(),
                    completedDate = null,
                ),
            ),
            3L to listOf(
                createStatusSla(
                    initiativeId = 3L,
                    statusId = 5L,
                    plannedDate = null,
                    completedDate = null,
                ),
            ),
        )

        val result = strategy.findMatchingInitiativeIds(
            candidateInitiativeIds = setOf(1L, 2L, 3L),
            session = session,
            context = context,
        )

        assertThat(result).containsExactlyInAnyOrder(1L, 3L)
    }

    private fun createStatus(
        id: Long,
        ordering: Long?,
    ): StatusEntity =
        mockk {
            every { this@mockk.id } returns id
            every { this@mockk.ordering } returns ordering
        }

    private fun createStatusSla(
        initiativeId: Long,
        statusId: Long,
        plannedDate: LocalDateTime?,
        completedDate: LocalDateTime?,
    ): AgentStatusSlaEntity =
        AgentStatusSlaEntity().apply {
            primaryKey.aiAgentId = initiativeId
            primaryKey.agentStatusId = statusId
            this.plannedDate = plannedDate
            this.completedDate = completedDate
        }
}
  ```
