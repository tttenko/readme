```java
@ExtendWith(MockKExtension::class)
class StatusSlaUpdaterTest {

    @MockK
    lateinit var agentStatusSlaRepository: AgentStatusSlaRepository

    @MockK
    lateinit var statusRepository: StatusRepository

    @MockK
    lateinit var messageProvider: MessageProvider

    private lateinit var statusSlaUpdater: StatusSlaUpdater

    @BeforeEach
    fun setUp() {
        statusSlaUpdater = StatusSlaUpdater(
            agentStatusSlaRepository = agentStatusSlaRepository,
            statusRepository = statusRepository,
            messageProvider = messageProvider,
        )

        every {
            messageProvider[WRONG_STATUS_SLA_VALUE]
        } returns "Некорректное значение statusSla"

        every {
            messageProvider[PROCESS_WITH_ID_NOT_FOUND]
        } returns "Значение {0} не найдено в справочнике"
    }

    @Test
    fun `update should delete all persisted status sla when raw value is null`() {
        val agent = agent(id = 1L)
        val persistedStatusSla = persistedStatusSla(
            statusId = 10L,
            plannedDate = LocalDateTime.of(2026, 5, 1, 10, 0),
        )

        every {
            agentStatusSlaRepository.findAllByAiAgentId(
                agentId = 1L,
            )
        } returns listOf(persistedStatusSla)

        every {
            agentStatusSlaRepository.deleteAll(
                entities = listOf(persistedStatusSla),
            )
        } just Runs

        val result = statusSlaUpdater.update(
            agent = agent,
            rawStatusSlaValue = null,
        )

        assertTrue(result.isEmpty())

        verify(exactly = 1) {
            agentStatusSlaRepository.deleteAll(
                entities = listOf(persistedStatusSla),
            )
        }

        verify(exactly = 0) {
            statusRepository.findAllByDisabledIsFalse()
        }
    }

    @Test
    fun `update should delete all persisted status sla when request list is empty`() {
        val agent = agent(id = 1L)
        val persistedStatusSla = persistedStatusSla(
            statusId = 10L,
            plannedDate = LocalDateTime.of(2026, 5, 1, 10, 0),
        )

        every {
            agentStatusSlaRepository.findAllByAiAgentId(
                agentId = 1L,
            )
        } returns listOf(persistedStatusSla)

        every {
            agentStatusSlaRepository.deleteAll(
                entities = listOf(persistedStatusSla),
            )
        } just Runs

        val result = statusSlaUpdater.update(
            agent = agent,
            rawStatusSlaValue = emptyList<Any>(),
        )

        assertTrue(result.isEmpty())

        verify(exactly = 1) {
            agentStatusSlaRepository.deleteAll(
                entities = listOf(persistedStatusSla),
            )
        }
    }

    @Test
    fun `update should create new status sla and return changed item`() {
        val agent = agent(id = 1L)
        val pilotStatus = status(
            id = 10L,
            code = "pilot",
        )

        val request = listOf(
            mapOf(
                "status" to "pilot",
                "plannedDate" to "2026-05-03",
            )
        )

        val savedSlot = slot<List<AgentStatusSlaEntity>>()

        every {
            agentStatusSlaRepository.findAllByAiAgentId(
                agentId = 1L,
            )
        } returns emptyList()

        every {
            statusRepository.findAllByDisabledIsFalse()
        } returns listOf(pilotStatus)

        every {
            agentStatusSlaRepository.saveAll(
                entities = capture(savedSlot),
            )
        } answers {
            savedSlot.captured
        }

        val result = statusSlaUpdater.update(
            agent = agent,
            rawStatusSlaValue = request,
        )

        assertEquals(1, result.size)
        assertEquals("pilot", result.first().status)
        assertEquals(LocalDate.of(2026, 5, 3), result.first().plannedDate)

        val savedEntity = savedSlot.captured.first()

        assertEquals(agent, savedEntity.aiAgent)
        assertEquals(pilotStatus, savedEntity.agentStatus)
        assertEquals(
            LocalDate.of(2026, 5, 3).atStartOfDay(),
            savedEntity.plannedDate,
        )

        verify(exactly = 1) {
            agentStatusSlaRepository.saveAll(
                entities = any<List<AgentStatusSlaEntity>>(),
            )
        }

        verify(exactly = 0) {
            agentStatusSlaRepository.deleteAll(
                entities = any<List<AgentStatusSlaEntity>>(),
            )
        }
    }

    @Test
    fun `update should update existing status sla when planned date changed`() {
        val agent = agent(id = 1L)
        val pilotStatus = status(
            id = 10L,
            code = "pilot",
        )

        val persistedStatusSla = persistedStatusSla(
            statusId = 10L,
            plannedDate = LocalDateTime.of(2026, 5, 1, 10, 0),
        )

        val request = listOf(
            mapOf(
                "status" to "pilot",
                "plannedDate" to "2026-05-03",
            )
        )

        every {
            agentStatusSlaRepository.findAllByAiAgentId(
                agentId = 1L,
            )
        } returns listOf(persistedStatusSla)

        every {
            statusRepository.findAllByDisabledIsFalse()
        } returns listOf(pilotStatus)

        every {
            persistedStatusSla.plannedDate =
                LocalDate.of(2026, 5, 3).atStartOfDay()
        } just Runs

        every {
            agentStatusSlaRepository.saveAll(
                entities = listOf(persistedStatusSla),
            )
        } returns listOf(persistedStatusSla)

        val result = statusSlaUpdater.update(
            agent = agent,
            rawStatusSlaValue = request,
        )

        assertEquals(1, result.size)
        assertEquals("pilot", result.first().status)
        assertEquals(LocalDate.of(2026, 5, 3), result.first().plannedDate)

        verify(exactly = 1) {
            persistedStatusSla.plannedDate =
                LocalDate.of(2026, 5, 3).atStartOfDay()
        }

        verify(exactly = 1) {
            agentStatusSlaRepository.saveAll(
                entities = listOf(persistedStatusSla),
            )
        }
    }

    @Test
    fun `update should not save when planned date did not change by local date`() {
        val agent = agent(id = 1L)
        val pilotStatus = status(
            id = 10L,
            code = "pilot",
        )

        val persistedStatusSla = persistedStatusSla(
            statusId = 10L,
            plannedDate = LocalDateTime.of(2026, 5, 3, 17, 30),
        )

        val request = listOf(
            mapOf(
                "status" to "pilot",
                "plannedDate" to "2026-05-03",
            )
        )

        every {
            agentStatusSlaRepository.findAllByAiAgentId(
                agentId = 1L,
            )
        } returns listOf(persistedStatusSla)

        every {
            statusRepository.findAllByDisabledIsFalse()
        } returns listOf(pilotStatus)

        val result = statusSlaUpdater.update(
            agent = agent,
            rawStatusSlaValue = request,
        )

        assertTrue(result.isEmpty())

        verify(exactly = 0) {
            agentStatusSlaRepository.saveAll(
                entities = any<List<AgentStatusSlaEntity>>(),
            )
        }

        verify(exactly = 0) {
            agentStatusSlaRepository.deleteAll(
                entities = any<List<AgentStatusSlaEntity>>(),
            )
        }
    }

    @Test
    fun `update should delete persisted status sla missing in request`() {
        val agent = agent(id = 1L)

        val pilotStatus = status(
            id = 10L,
            code = "pilot",
        )

        val persistedPilotSla = persistedStatusSla(
            statusId = 10L,
            plannedDate = LocalDateTime.of(2026, 5, 3, 10, 0),
        )

        val persistedTargetSla = persistedStatusSla(
            statusId = 20L,
            plannedDate = LocalDateTime.of(2026, 6, 3, 10, 0),
        )

        val request = listOf(
            mapOf(
                "status" to "pilot",
                "plannedDate" to "2026-05-03",
            )
        )

        every {
            agentStatusSlaRepository.findAllByAiAgentId(
                agentId = 1L,
            )
        } returns listOf(
            persistedPilotSla,
            persistedTargetSla,
        )

        every {
            statusRepository.findAllByDisabledIsFalse()
        } returns listOf(pilotStatus)

        every {
            agentStatusSlaRepository.deleteAll(
                entities = listOf(persistedTargetSla),
            )
        } just Runs

        val result = statusSlaUpdater.update(
            agent = agent,
            rawStatusSlaValue = request,
        )

        assertTrue(result.isEmpty())

        verify(exactly = 1) {
            agentStatusSlaRepository.deleteAll(
                entities = listOf(persistedTargetSla),
            )
        }

        verify(exactly = 0) {
            agentStatusSlaRepository.saveAll(
                entities = any<List<AgentStatusSlaEntity>>(),
            )
        }
    }

    @Test
    fun `update should parse LocalDate planned date`() {
        val agent = agent(id = 1L)
        val pilotStatus = status(
            id = 10L,
            code = "pilot",
        )

        val plannedDate = LocalDate.of(2026, 5, 3)

        val request = listOf(
            mapOf(
                "status" to "pilot",
                "plannedDate" to plannedDate,
            )
        )

        every {
            agentStatusSlaRepository.findAllByAiAgentId(
                agentId = 1L,
            )
        } returns emptyList()

        every {
            statusRepository.findAllByDisabledIsFalse()
        } returns listOf(pilotStatus)

        every {
            agentStatusSlaRepository.saveAll(
                entities = any<List<AgentStatusSlaEntity>>(),
            )
        } answers {
            firstArg()
        }

        val result = statusSlaUpdater.update(
            agent = agent,
            rawStatusSlaValue = request,
        )

        assertEquals(1, result.size)
        assertEquals("pilot", result.first().status)
        assertEquals(plannedDate, result.first().plannedDate)
    }

    @Test
    fun `update should throw exception when raw status sla is not list`() {
        val agent = agent(id = 1L)

        val exception = assertThrows<AiBadRequestException> {
            statusSlaUpdater.update(
                agent = agent,
                rawStatusSlaValue = "wrong value",
            )
        }

        assertEquals(WRONG_STATUS_SLA_VALUE, exception.errorCode)
        assertEquals("Некорректное значение statusSla", exception.message)
    }

    @Test
    fun `update should throw exception when item is not map`() {
        val agent = agent(id = 1L)

        val exception = assertThrows<AiBadRequestException> {
            statusSlaUpdater.update(
                agent = agent,
                rawStatusSlaValue = listOf("wrong item"),
            )
        }

        assertEquals(WRONG_STATUS_SLA_VALUE, exception.errorCode)
    }

    @Test
    fun `update should throw exception when status is blank`() {
        val agent = agent(id = 1L)

        val request = listOf(
            mapOf(
                "status" to "   ",
                "plannedDate" to "2026-05-03",
            )
        )

        val exception = assertThrows<AiBadRequestException> {
            statusSlaUpdater.update(
                agent = agent,
                rawStatusSlaValue = request,
            )
        }

        assertEquals(WRONG_STATUS_SLA_VALUE, exception.errorCode)
    }

    @Test
    fun `update should throw exception when planned date has wrong format`() {
        val agent = agent(id = 1L)

        val request = listOf(
            mapOf(
                "status" to "pilot",
                "plannedDate" to "03.05.2026",
            )
        )

        val exception = assertThrows<AiBadRequestException> {
            statusSlaUpdater.update(
                agent = agent,
                rawStatusSlaValue = request,
            )
        }

        assertEquals(WRONG_STATUS_SLA_VALUE, exception.errorCode)
    }

    @Test
    fun `update should throw exception when requested status does not exist`() {
        val agent = agent(id = 1L)

        val request = listOf(
            mapOf(
                "status" to "unknown",
                "plannedDate" to "2026-05-03",
            )
        )

        every {
            agentStatusSlaRepository.findAllByAiAgentId(
                agentId = 1L,
            )
        } returns emptyList()

        every {
            statusRepository.findAllByDisabledIsFalse()
        } returns emptyList()

        val exception = assertThrows<AiBadRequestException> {
            statusSlaUpdater.update(
                agent = agent,
                rawStatusSlaValue = request,
            )
        }

        assertEquals(PROCESS_WITH_ID_NOT_FOUND, exception.errorCode)
        assertEquals(
            "Значение unknown не найдено в справочнике",
            exception.message,
        )
    }

    private fun agent(
        id: Long,
    ): AIAgentEntity {
        return mockk<AIAgentEntity>(relaxed = true) {
            every {
                this@mockk.id
            } returns id
        }
    }

    private fun status(
        id: Long,
        code: String,
    ): StatusEntity {
        return mockk<StatusEntity>(relaxed = true) {
            every {
                this@mockk.id
            } returns id

            every {
                this@mockk.code
            } returns code
        }
    }

    private fun persistedStatusSla(
        statusId: Long,
        plannedDate: LocalDateTime?,
    ): AgentStatusSlaEntity {
        return mockk<AgentStatusSlaEntity>(relaxed = true) {
            every {
                this@mockk.primaryKey.agentStatusId
            } returns statusId

            every {
                this@mockk.plannedDate
            } returns plannedDate

            every {
                this@mockk.plannedDate = any()
            } just Runs
        }
    }
}
  
```
