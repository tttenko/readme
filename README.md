```java
@ExtendWith(MockKExtension::class)
 class InitiativeDeviationCalculationSessionTest {

    @MockK
    private lateinit var agentStatusSlaRepository: AgentStatusSlaRepository

    @MockK
    private lateinit var jiraIssueRepository: JiraIssueRepository

    @MockK
    private lateinit var enablerRepository: EnablerRepository

    @MockK
    private lateinit var aiAgentRepository: AIAgentRepository

    @MockK
    private lateinit var currentPeriodMetricsDeviationFinder:
        CurrentPeriodMetricsDeviationFinder


    @Test
    fun `statusSlaByInitiativeId should return empty map and not call repository when initiative ids are empty`() {
        val session = createSession(
            initiativeIds = emptySet()
        )

        val result = session.statusSlaByInitiativeId

        assertThat(result).isEmpty()

        verify(exactly = 0) {
            agentStatusSlaRepository.findAllByAiAgentIdIn(any())
        }
    }

    @Test
    fun `statusSlaByInitiativeId should group SLA records by initiative id`() {
        val firstAgent = createAgent(id = 1L)
        val secondAgent = createAgent(id = 2L)

        val firstSla = createStatusSla(agent = firstAgent)
        val secondSla = createStatusSla(agent = firstAgent)
        val thirdSla = createStatusSla(agent = secondAgent)

        every {
            agentStatusSlaRepository.findAllByAiAgentIdIn(
                setOf(1L, 2L)
            )
        } returns listOf(
            firstSla,
            secondSla,
            thirdSla
        )

        val session = createSession(
            initiativeIds = setOf(1L, 2L)
        )

        val result = session.statusSlaByInitiativeId

        assertThat(result.keys)
            .containsExactlyInAnyOrder(1L, 2L)

        assertThat(result.getValue(1L))
            .containsExactly(firstSla, secondSla)

        assertThat(result.getValue(2L))
            .containsExactly(thirdSla)

        verify(exactly = 1) {
            agentStatusSlaRepository.findAllByAiAgentIdIn(
                setOf(1L, 2L)
            )
        }
    }

    @Test
    fun `statusSlaByInitiativeId should ignore records without initiative`() {
        val agent = createAgent(id = 1L)

        val validSla = createStatusSla(agent = agent)
        val slaWithoutAgent = createStatusSla(agent = null)

        every {
            agentStatusSlaRepository.findAllByAiAgentIdIn(
                setOf(1L)
            )
        } returns listOf(
            validSla,
            slaWithoutAgent
        )

        val session = createSession(
            initiativeIds = setOf(1L)
        )

        val result = session.statusSlaByInitiativeId

        assertThat(result)
            .containsOnlyKeys(1L)

        assertThat(result.getValue(1L))
            .containsExactly(validSla)
    }

    @Test
    fun `statusSlaByInitiativeId should ignore records whose initiative id is null`() {
        val agentWithoutId = mockk<AIAgentEntity>()

        every {
            agentWithoutId.id
        } returns null

        val sla = createStatusSla(
            agent = agentWithoutId
        )

        every {
            agentStatusSlaRepository.findAllByAiAgentIdIn(
                setOf(1L)
            )
        } returns listOf(sla)

        val session = createSession(
            initiativeIds = setOf(1L)
        )

        val result = session.statusSlaByInitiativeId

        assertThat(result).isEmpty()
    }

    @Test
    fun `statusSlaByInitiativeId should call repository only once on repeated access`() {
        every {
            agentStatusSlaRepository.findAllByAiAgentIdIn(
                setOf(1L)
            )
        } returns emptyList()

        val session = createSession(
            initiativeIds = setOf(1L)
        )

        session.statusSlaByInitiativeId
        session.statusSlaByInitiativeId
        session.statusSlaByInitiativeId

        verify(exactly = 1) {
            agentStatusSlaRepository.findAllByAiAgentIdIn(
                setOf(1L)
            )
        }
    }

    @Test
    fun `initiativeIdsWithValidGigaUsage should return empty set and not call repository when initiative ids are empty`() {
        val session = createSession(
            initiativeIds = emptySet()
        )

        val result =
            session.initiativeIdsWithValidGigaUsage

        assertThat(result).isEmpty()

        verify(exactly = 0) {
            jiraIssueRepository
                .findInitiativeIdsWithValidGigaUsage(any())
        }
    }

    @Test
    fun `initiativeIdsWithValidGigaUsage should return initiative ids from repository`() {
        every {
            jiraIssueRepository
                .findInitiativeIdsWithValidGigaUsage(
                    setOf(1L, 2L, 3L)
                )
        } returns setOf(1L, 3L)

        val session = createSession(
            initiativeIds = setOf(1L, 2L, 3L)
        )

        val result =
            session.initiativeIdsWithValidGigaUsage

        assertThat(result)
            .containsExactlyInAnyOrder(1L, 3L)

        verify(exactly = 1) {
            jiraIssueRepository
                .findInitiativeIdsWithValidGigaUsage(
                    setOf(1L, 2L, 3L)
                )
        }
    }

    @Test
    fun `initiativeIdsWithValidGigaUsage should call repository only once on repeated access`() {
        every {
            jiraIssueRepository
                .findInitiativeIdsWithValidGigaUsage(
                    setOf(1L)
                )
        } returns setOf(1L)

        val session = createSession(
            initiativeIds = setOf(1L)
        )

        session.initiativeIdsWithValidGigaUsage
        session.initiativeIdsWithValidGigaUsage

        verify(exactly = 1) {
            jiraIssueRepository
                .findInitiativeIdsWithValidGigaUsage(
                    setOf(1L)
                )
        }
    }

    @Test
    fun `initiativeIdsWithEnablers should return empty set and not call repository when initiative ids are empty`() {
        val session = createSession(
            initiativeIds = emptySet()
        )

        val result = session.initiativeIdsWithEnablers

        assertThat(result).isEmpty()

        verify(exactly = 0) {
            enablerRepository
                .findInitiativeIdsWithEnablers(any())
        }
    }

    @Test
    fun `initiativeIdsWithEnablers should return initiative ids from repository`() {
        every {
            enablerRepository
                .findInitiativeIdsWithEnablers(
                    setOf(1L, 2L, 3L)
                )
        } returns setOf(2L, 3L)

        val session = createSession(
            initiativeIds = setOf(1L, 2L, 3L)
        )

        val result = session.initiativeIdsWithEnablers

        assertThat(result)
            .containsExactlyInAnyOrder(2L, 3L)

        verify(exactly = 1) {
            enablerRepository
                .findInitiativeIdsWithEnablers(
                    setOf(1L, 2L, 3L)
                )
        }
    }

    @Test
    fun `initiativeIdsWithEnablers should call repository only once on repeated access`() {
        every {
            enablerRepository
                .findInitiativeIdsWithEnablers(
                    setOf(1L)
                )
        } returns setOf(1L)

        val session = createSession(
            initiativeIds = setOf(1L)
        )

        session.initiativeIdsWithEnablers
        session.initiativeIdsWithEnablers

        verify(exactly = 1) {
            enablerRepository
                .findInitiativeIdsWithEnablers(
                    setOf(1L)
                )
        }
    }

    @Test
    fun `statusCodeByInitiativeId should return empty map and not call repository when initiative ids are empty`() {
        val session = createSession(
            initiativeIds = emptySet()
        )

        val result = session.statusCodeByInitiativeId

        assertThat(result).isEmpty()

        verify(exactly = 0) {
            aiAgentRepository.findInitiativeStatuses(any())
        }
    }

    @Test
    fun `statusCodeByInitiativeId should map initiative ids to status codes`() {
        val firstProjection =
            createStatusProjection(
                initiativeId = 1L,
                statusCode = "targetSolution"
            )

        val secondProjection =
            createStatusProjection(
                initiativeId = 2L,
                statusCode = "implementation"
            )

        every {
            aiAgentRepository.findInitiativeStatuses(
                setOf(1L, 2L)
            )
        } returns listOf(
            firstProjection,
            secondProjection
        )

        val session = createSession(
            initiativeIds = setOf(1L, 2L)
        )

        val result = session.statusCodeByInitiativeId

        assertThat(result)
            .containsEntry(1L, "targetSolution")
            .containsEntry(2L, "implementation")
            .hasSize(2)

        verify(exactly = 1) {
            aiAgentRepository.findInitiativeStatuses(
                setOf(1L, 2L)
            )
        }
    }

    @Test
    fun `statusCodeByInitiativeId should preserve null status code`() {
        val projection =
            createStatusProjection(
                initiativeId = 1L,
                statusCode = null
            )

        every {
            aiAgentRepository.findInitiativeStatuses(
                setOf(1L)
            )
        } returns listOf(projection)

        val session = createSession(
            initiativeIds = setOf(1L)
        )

        val result = session.statusCodeByInitiativeId

        assertThat(result)
            .containsKey(1L)

        assertThat(result[1L]).isNull()
    }

    @Test
    fun `statusCodeByInitiativeId should use last status when repository returns duplicate initiative ids`() {
        val firstProjection =
            createStatusProjection(
                initiativeId = 1L,
                statusCode = "draft"
            )

        val secondProjection =
            createStatusProjection(
                initiativeId = 1L,
                statusCode = "targetSolution"
            )

        every {
            aiAgentRepository.findInitiativeStatuses(
                setOf(1L)
            )
        } returns listOf(
            firstProjection,
            secondProjection
        )

        val session = createSession(
            initiativeIds = setOf(1L)
        )

        val result = session.statusCodeByInitiativeId

        assertThat(result)
            .containsEntry(1L, "targetSolution")
            .hasSize(1)
    }

    @Test
    fun `statusCodeByInitiativeId should call repository only once on repeated access`() {
        every {
            aiAgentRepository.findInitiativeStatuses(
                setOf(1L)
            )
        } returns emptyList()

        val session = createSession(
            initiativeIds = setOf(1L)
        )

        session.statusCodeByInitiativeId
        session.statusCodeByInitiativeId
        session.statusCodeByInitiativeId

        verify(exactly = 1) {
            aiAgentRepository.findInitiativeStatuses(
                setOf(1L)
            )
        }
    }

    @Test
    fun `accessing one lazy property should not load data for other properties`() {
        every {
            jiraIssueRepository
                .findInitiativeIdsWithValidGigaUsage(
                    setOf(1L)
                )
        } returns setOf(1L)

        val session = createSession(
            initiativeIds = setOf(1L)
        )

        val result =
            session.initiativeIdsWithValidGigaUsage

        assertThat(result).containsExactly(1L)

        verify(exactly = 1) {
            jiraIssueRepository
                .findInitiativeIdsWithValidGigaUsage(
                    setOf(1L)
                )
        }

        verify(exactly = 0) {
            agentStatusSlaRepository
                .findAllByAiAgentIdIn(any())
        }

        verify(exactly = 0) {
            enablerRepository
                .findInitiativeIdsWithEnablers(any())
        }

        verify(exactly = 0) {
            aiAgentRepository
                .findInitiativeStatuses(any())
        }
    }

    @Test
    fun `session should expose provided metrics deviation finder`() {
        val session = createSession(
            initiativeIds = setOf(1L)
        )

        assertThat(
            session.currentPeriodMetricsDeviationFinder
        ).isSameAs(currentPeriodMetricsDeviationFinder)
    }

    private fun createSession(
        initiativeIds: Set<Long>
    ): InitiativeDeviationCalculationSession =
        InitiativeDeviationCalculationSession(
            initiativeIds = initiativeIds,
            agentStatusSlaRepository =
                agentStatusSlaRepository,
            jiraIssueRepository =
                jiraIssueRepository,
            enablerRepository =
                enablerRepository,
            aiAgentRepository =
                aiAgentRepository,
            currentPeriodMetricsDeviationFinder =
                currentPeriodMetricsDeviationFinder
        )

    private fun createAgent(
        id: Long
    ): AIAgentEntity {
        val agent = mockk<AIAgentEntity>()

        every {
            agent.id
        } returns id

        return agent
    }

    private fun createStatusSla(
        agent: AIAgentEntity?
    ): AgentStatusSlaEntity {
        val statusSla = mockk<AgentStatusSlaEntity>()

        every {
            statusSla.aiAgent
        } returns agent

        return statusSla
    }

    private fun createStatusProjection(
        initiativeId: Long,
        statusCode: String?
    ): InitiativeStatusProjection {
        val projection =
            mockk<InitiativeStatusProjection>()

        every {
            projection.initiativeId
        } returns initiativeId

        every {
            projection.statusCode
        } returns statusCode

        return projection
    }
}

```
