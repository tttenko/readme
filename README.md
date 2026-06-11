```java
@ExtendWith(MockKExtension::class)
class JiraChangeCreatorTest {

    @MockK
    lateinit var jiraIssueRepository: JiraIssueRepository

    @MockK
    lateinit var jiraChangeRepository: JiraChangeRepository

    private lateinit var objectMapper: ObjectMapper

    private lateinit var jiraChangeCreator: JiraChangeCreator

    @BeforeEach
    fun setUp() {
        objectMapper = ObjectMapper()
            .registerModule(JavaTimeModule())
            .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)

        jiraChangeCreator = JiraChangeCreator(
            jiraIssueRepository = jiraIssueRepository,
            jiraChangeRepository = jiraChangeRepository,
            objectMapper = objectMapper,
        )
    }

    @Test
    fun `createChanges should do nothing when agent has no crossgoal initiative`() {
        val agent = agent(id = 1L)

        every {
            jiraIssueRepository.existsByAgentIdAndTypeAndProject(
                agentId = 1L,
                type = "initiative",
                project = "crossgoal",
            )
        } returns false

        jiraChangeCreator.createChanges(
            agent = agent,
            request = mapOf(
                "agentName" to "Test agent",
            ),
            changedStatusSla = emptyList(),
            userId = 100L,
        )

        verify(exactly = 0) {
            jiraChangeRepository.saveAll(
                entities = any<List<JiraChangeEntity>>(),
            )
        }

        verify(exactly = 0) {
            agent.jiraStatus = any()
        }

        verify(exactly = 0) {
            agent.updatedBy = any()
        }
    }

    @Test
    fun `createChanges should do nothing when crossgoal exists but request has no jira fields and status sla is empty`() {
        val agent = agent(id = 1L)

        every {
            jiraIssueRepository.existsByAgentIdAndTypeAndProject(
                agentId = 1L,
                type = "initiative",
                project = "crossgoal",
            )
        } returns true

        jiraChangeCreator.createChanges(
            agent = agent,
            request = mapOf(
                "platforms" to emptyList<Any>(),
                "jiraStatus" to "done",
            ),
            changedStatusSla = emptyList(),
            userId = 100L,
        )

        verify(exactly = 0) {
            jiraChangeRepository.saveAll(
                entities = any<List<JiraChangeEntity>>(),
            )
        }

        verify(exactly = 0) {
            agent.jiraStatus = any()
        }

        verify(exactly = 0) {
            agent.updatedBy = any()
        }
    }

    @Test
    fun `createChanges should create initiative change when request contains jira sync fields`() {
        val agent = agent(id = 1L)
        val savedChangesSlot = slot<List<JiraChangeEntity>>()

        val request: Map<String, Any?> = mapOf(
            "agentName" to "AI Agent",
            "agentDescription" to null,
            "agentEffectRevenue" to 100,
            "platforms" to emptyList<Any>(),
            "jiraStatus" to "done",
        )

        every {
            jiraIssueRepository.existsByAgentIdAndTypeAndProject(
                agentId = 1L,
                type = "initiative",
                project = "crossgoal",
            )
        } returns true

        every {
            jiraChangeRepository.saveAll(
                entities = capture(savedChangesSlot),
            )
        } answers {
            savedChangesSlot.captured
        }

        every {
            agent.jiraStatus = "pendingUpdate"
        } just Runs

        every {
            agent.updated = any()
        } just Runs

        every {
            agent.updatedBy = 100L
        } just Runs

        jiraChangeCreator.createChanges(
            agent = agent,
            request = request,
            changedStatusSla = emptyList(),
            userId = 100L,
        )

        val savedChanges = savedChangesSlot.captured

        assertEquals(1, savedChanges.size)

        val initiativeChange = savedChanges.first()

        assertEquals(agent, initiativeChange.agent)
        assertEquals("initiative", initiativeChange.changeType)

        assertEquals(
            "AI Agent",
            initiativeChange.payload["agentName"].asText(),
        )

        assertTrue(
            initiativeChange.payload.has("agentDescription"),
        )

        assertTrue(
            initiativeChange.payload["agentDescription"].isNull,
        )

        assertEquals(
            100,
            initiativeChange.payload["agentEffectRevenue"].asInt(),
        )

        assertFalse(
            initiativeChange.payload.has("platforms"),
        )

        assertFalse(
            initiativeChange.payload.has("jiraStatus"),
        )

        verify(exactly = 1) {
            agent.jiraStatus = "pendingUpdate"
        }

        verify(exactly = 1) {
            agent.updatedBy = 100L
        }
    }

    @Test
    fun `createChanges should create plannedDate change when changedStatusSla is not empty`() {
        val agent = agent(id = 1L)
        val savedChangesSlot = slot<List<JiraChangeEntity>>()

        val changedStatusSla = listOf(
            StatusSlaDto(
                status = "pilot",
                plannedDate = LocalDate.of(2026, 5, 3),
            )
        )

        every {
            jiraIssueRepository.existsByAgentIdAndTypeAndProject(
                agentId = 1L,
                type = "initiative",
                project = "crossgoal",
            )
        } returns true

        every {
            jiraChangeRepository.saveAll(
                entities = capture(savedChangesSlot),
            )
        } answers {
            savedChangesSlot.captured
        }

        every {
            agent.jiraStatus = "pendingUpdate"
        } just Runs

        every {
            agent.updated = any()
        } just Runs

        every {
            agent.updatedBy = 100L
        } just Runs

        jiraChangeCreator.createChanges(
            agent = agent,
            request = emptyMap(),
            changedStatusSla = changedStatusSla,
            userId = 100L,
        )

        val savedChanges = savedChangesSlot.captured

        assertEquals(1, savedChanges.size)

        val plannedDateChange = savedChanges.first()

        assertEquals(agent, plannedDateChange.agent)
        assertEquals("plannedDate", plannedDateChange.changeType)

        val statusSlaNode =
            plannedDateChange.payload["statusSla"]

        assertEquals(1, statusSlaNode.size())

        assertEquals(
            "pilot",
            statusSlaNode[0]["status"].asText(),
        )

        assertEquals(
            "2026-05-03T00:00:00Z",
            statusSlaNode[0]["plannedDate"].asText(),
        )

        verify(exactly = 1) {
            agent.jiraStatus = "pendingUpdate"
        }

        verify(exactly = 1) {
            agent.updatedBy = 100L
        }
    }

    @Test
    fun `createChanges should create initiative and plannedDate changes together`() {
        val agent = agent(id = 1L)
        val savedChangesSlot = slot<List<JiraChangeEntity>>()

        val request: Map<String, Any?> = mapOf(
            "agentName" to "AI Agent",
            "enablers" to listOf(1, 2),
        )

        val changedStatusSla = listOf(
            StatusSlaDto(
                status = "pilot",
                plannedDate = LocalDate.of(2026, 5, 3),
            )
        )

        every {
            jiraIssueRepository.existsByAgentIdAndTypeAndProject(
                agentId = 1L,
                type = "initiative",
                project = "crossgoal",
            )
        } returns true

        every {
            jiraChangeRepository.saveAll(
                entities = capture(savedChangesSlot),
            )
        } answers {
            savedChangesSlot.captured
        }

        every {
            agent.jiraStatus = "pendingUpdate"
        } just Runs

        every {
            agent.updated = any()
        } just Runs

        every {
            agent.updatedBy = 100L
        } just Runs

        jiraChangeCreator.createChanges(
            agent = agent,
            request = request,
            changedStatusSla = changedStatusSla,
            userId = 100L,
        )

        val savedChanges = savedChangesSlot.captured

        assertEquals(2, savedChanges.size)

        val initiativeChange =
            savedChanges.first { jiraChange ->
                jiraChange.changeType == "initiative"
            }

        val plannedDateChange =
            savedChanges.first { jiraChange ->
                jiraChange.changeType == "plannedDate"
            }

        assertEquals(
            "AI Agent",
            initiativeChange.payload["agentName"].asText(),
        )

        assertEquals(
            1,
            initiativeChange.payload["enablers"][0].asInt(),
        )

        assertEquals(
            2,
            initiativeChange.payload["enablers"][1].asInt(),
        )

        assertEquals(
            "pilot",
            plannedDateChange.payload["statusSla"][0]["status"].asText(),
        )

        verify(exactly = 1) {
            jiraChangeRepository.saveAll(
                entities = any<List<JiraChangeEntity>>(),
            )
        }

        verify(exactly = 1) {
            agent.jiraStatus = "pendingUpdate"
        }

        verify(exactly = 1) {
            agent.updatedBy = 100L
        }
    }

    @Test
    fun `createChanges should keep only initiative sync fields in initiative payload`() {
        val agent = agent(id = 1L)
        val savedChangesSlot = slot<List<JiraChangeEntity>>()

        val request: Map<String, Any?> = mapOf(
            "agentName" to "AI Agent",
            "agentDescription" to "Description",
            "agentInitiativeType" to "agent",
            "block" to "cib",
            "division" to "DEVU",
            "strategies" to listOf(1),
            "processes" to listOf(2),
            "enablers" to listOf(3),
            "involvedResources" to listOf(
                mapOf(
                    "value" to 10,
                    "source" to "steerco",
                    "type" to "business",
                )
            ),
            "agentEffectOptimization" to 100,
            "agentEffectRevenue" to 200,
            "platforms" to listOf("web"),
            "statusSla" to emptyList<Any>(),
            "jiraStatus" to "done",
            "gigausage" to listOf("GIGAUSAGE-1"),
        )

        every {
            jiraIssueRepository.existsByAgentIdAndTypeAndProject(
                agentId = 1L,
                type = "initiative",
                project = "crossgoal",
            )
        } returns true

        every {
            jiraChangeRepository.saveAll(
                entities = capture(savedChangesSlot),
            )
        } answers {
            savedChangesSlot.captured
        }

        every {
            agent.jiraStatus = "pendingUpdate"
        } just Runs

        every {
            agent.updated = any()
        } just Runs

        every {
            agent.updatedBy = 100L
        } just Runs

        jiraChangeCreator.createChanges(
            agent = agent,
            request = request,
            changedStatusSla = emptyList(),
            userId = 100L,
        )

        val payload =
            savedChangesSlot.captured.first().payload

        assertTrue(payload.has("agentName"))
        assertTrue(payload.has("agentDescription"))
        assertTrue(payload.has("agentInitiativeType"))
        assertTrue(payload.has("block"))
        assertTrue(payload.has("division"))
        assertTrue(payload.has("strategies"))
        assertTrue(payload.has("processes"))
        assertTrue(payload.has("enablers"))
        assertTrue(payload.has("involvedResources"))
        assertTrue(payload.has("agentEffectOptimization"))
        assertTrue(payload.has("agentEffectRevenue"))

        assertFalse(payload.has("platforms"))
        assertFalse(payload.has("statusSla"))
        assertFalse(payload.has("jiraStatus"))
        assertFalse(payload.has("gigausage"))
    }

    @Test
    fun `createChanges should throw exception when changed status sla has null status`() {
        val agent = agent(id = 1L)

        val changedStatusSla = listOf(
            StatusSlaDto(
                status = null,
                plannedDate = LocalDate.of(2026, 5, 3),
            )
        )

        every {
            jiraIssueRepository.existsByAgentIdAndTypeAndProject(
                agentId = 1L,
                type = "initiative",
                project = "crossgoal",
            )
        } returns true

        assertThrows<IllegalArgumentException> {
            jiraChangeCreator.createChanges(
                agent = agent,
                request = emptyMap(),
                changedStatusSla = changedStatusSla,
                userId = 100L,
            )
        }

        verify(exactly = 0) {
            jiraChangeRepository.saveAll(
                entities = any<List<JiraChangeEntity>>(),
            )
        }
    }

    @Test
    fun `createChanges should throw exception when changed status sla has null plannedDate`() {
        val agent = agent(id = 1L)

        val changedStatusSla = listOf(
            StatusSlaDto(
                status = "pilot",
                plannedDate = null,
            )
        )

        every {
            jiraIssueRepository.existsByAgentIdAndTypeAndProject(
                agentId = 1L,
                type = "initiative",
                project = "crossgoal",
            )
        } returns true

        assertThrows<IllegalArgumentException> {
            jiraChangeCreator.createChanges(
                agent = agent,
                request = emptyMap(),
                changedStatusSla = changedStatusSla,
                userId = 100L,
            )
        }

        verify(exactly = 0) {
            jiraChangeRepository.saveAll(
                entities = any<List<JiraChangeEntity>>(),
            )
        }
    }

    private fun agent(
        id: Long,
    ): AIAgentEntity {
        return mockk<AIAgentEntity>(relaxed = true) {
            every {
                this@mockk.id
            } returns id

            every {
                this@mockk.jiraStatus = any()
            } just Runs

            every {
                this@mockk.updated = any()
            } just Runs

            every {
                this@mockk.updatedBy = any()
            } just Runs
        }
    }
}
  
```
