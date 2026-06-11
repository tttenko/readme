```java
@ExtendWith(MockKExtension::class)
class AgentEnablerUpdaterTest {

    @MockK
    lateinit var enablerRepository: EnablerRepository

    @MockK
    lateinit var messageProvider: MessageProvider

    private lateinit var agentEnablerUpdater: AgentEnablerUpdater

    @BeforeEach
    fun setUp() {
        agentEnablerUpdater = AgentEnablerUpdater(
            enablerRepository = enablerRepository,
            messageProvider = messageProvider,
        )

        every {
            messageProvider[WRONG_ENABLERS_VALUE]
        } returns "Некорректное значение enablers"

        every {
            messageProvider[ENABLER_NOT_FOUND]
        } returns "Значение {0} не найдено в справочнике"
    }

    @Test
    fun `updateEnablers should clear enablers when raw value is null`() {
        val existingEnabler = enabler(id = 1L)
        val agentEnablers = mutableSetOf(existingEnabler)
        val agent = agent(enablers = agentEnablers)

        agentEnablerUpdater.updateEnablers(
            agent = agent,
            rawEnablersValue = null,
        )

        assertTrue(agentEnablers.isEmpty())

        verify(exactly = 0) {
            enablerRepository.findAllById(any<Iterable<Long>>())
        }
    }

    @Test
    fun `updateEnablers should clear enablers when raw value is empty list`() {
        val existingEnabler = enabler(id = 1L)
        val agentEnablers = mutableSetOf(existingEnabler)
        val agent = agent(enablers = agentEnablers)

        agentEnablerUpdater.updateEnablers(
            agent = agent,
            rawEnablersValue = emptyList<Long>(),
        )

        assertTrue(agentEnablers.isEmpty())

        verify(exactly = 0) {
            enablerRepository.findAllById(any<Iterable<Long>>())
        }
    }

    @Test
    fun `updateEnablers should replace current enablers with requested enablers`() {
        val oldEnabler = enabler(id = 1L)
        val requestedEnabler1 = enabler(id = 2L)
        val requestedEnabler2 = enabler(id = 3L)

        val agentEnablers = mutableSetOf(oldEnabler)
        val agent = agent(enablers = agentEnablers)

        every {
            enablerRepository.findAllById(
                match<Iterable<Long>> { ids ->
                    ids.toSet() == setOf(2L, 3L)
                }
            )
        } returns listOf(
            requestedEnabler1,
            requestedEnabler2,
        )

        agentEnablerUpdater.updateEnablers(
            agent = agent,
            rawEnablersValue = listOf(2, 3),
        )

        assertEquals(
            setOf(requestedEnabler1, requestedEnabler2),
            agentEnablers,
        )

        verify(exactly = 1) {
            enablerRepository.findAllById(
                match<Iterable<Long>> { ids ->
                    ids.toSet() == setOf(2L, 3L)
                }
            )
        }
    }

    @Test
    fun `updateEnablers should convert duplicate ids to unique set`() {
        val requestedEnabler1 = enabler(id = 1L)
        val requestedEnabler2 = enabler(id = 2L)

        val agentEnablers = mutableSetOf<EnablerEntity>()
        val agent = agent(enablers = agentEnablers)

        every {
            enablerRepository.findAllById(
                match<Iterable<Long>> { ids ->
                    ids.toSet() == setOf(1L, 2L)
                }
            )
        } returns listOf(
            requestedEnabler1,
            requestedEnabler2,
        )

        agentEnablerUpdater.updateEnablers(
            agent = agent,
            rawEnablersValue = listOf(1, 1, 2),
        )

        assertEquals(2, agentEnablers.size)
        assertTrue(agentEnablers.contains(requestedEnabler1))
        assertTrue(agentEnablers.contains(requestedEnabler2))

        verify(exactly = 1) {
            enablerRepository.findAllById(
                match<Iterable<Long>> { ids ->
                    ids.toSet() == setOf(1L, 2L)
                }
            )
        }
    }

    @Test
    fun `updateEnablers should throw exception when raw value is not list`() {
        val existingEnabler = enabler(id = 1L)
        val agentEnablers = mutableSetOf(existingEnabler)
        val agent = agent(enablers = agentEnablers)

        val exception = assertThrows<AiBadRequestException> {
            agentEnablerUpdater.updateEnablers(
                agent = agent,
                rawEnablersValue = "wrong value",
            )
        }

        assertEquals(WRONG_ENABLERS_VALUE, exception.errorCode)
        assertEquals("Некорректное значение enablers", exception.message)

        assertEquals(setOf(existingEnabler), agentEnablers)

        verify(exactly = 0) {
            enablerRepository.findAllById(any<Iterable<Long>>())
        }
    }

    @Test
    fun `updateEnablers should throw exception when list item is not number`() {
        val existingEnabler = enabler(id = 1L)
        val agentEnablers = mutableSetOf(existingEnabler)
        val agent = agent(enablers = agentEnablers)

        val exception = assertThrows<AiBadRequestException> {
            agentEnablerUpdater.updateEnablers(
                agent = agent,
                rawEnablersValue = listOf("1"),
            )
        }

        assertEquals(WRONG_ENABLERS_VALUE, exception.errorCode)
        assertEquals(setOf(existingEnabler), agentEnablers)

        verify(exactly = 0) {
            enablerRepository.findAllById(any<Iterable<Long>>())
        }
    }

    @Test
    fun `updateEnablers should throw exception when enabler id is zero`() {
        val existingEnabler = enabler(id = 1L)
        val agentEnablers = mutableSetOf(existingEnabler)
        val agent = agent(enablers = agentEnablers)

        val exception = assertThrows<AiBadRequestException> {
            agentEnablerUpdater.updateEnablers(
                agent = agent,
                rawEnablersValue = listOf(0),
            )
        }

        assertEquals(WRONG_ENABLERS_VALUE, exception.errorCode)
        assertEquals(setOf(existingEnabler), agentEnablers)

        verify(exactly = 0) {
            enablerRepository.findAllById(any<Iterable<Long>>())
        }
    }

    @Test
    fun `updateEnablers should throw exception when enabler id is negative`() {
        val existingEnabler = enabler(id = 1L)
        val agentEnablers = mutableSetOf(existingEnabler)
        val agent = agent(enablers = agentEnablers)

        val exception = assertThrows<AiBadRequestException> {
            agentEnablerUpdater.updateEnablers(
                agent = agent,
                rawEnablersValue = listOf(-1),
            )
        }

        assertEquals(WRONG_ENABLERS_VALUE, exception.errorCode)
        assertEquals(setOf(existingEnabler), agentEnablers)

        verify(exactly = 0) {
            enablerRepository.findAllById(any<Iterable<Long>>())
        }
    }

    @Test
    fun `updateEnablers should throw exception when requested enabler does not exist`() {
        val existingAgentEnabler = enabler(id = 10L)
        val foundEnabler = enabler(id = 1L)

        val agentEnablers = mutableSetOf(existingAgentEnabler)
        val agent = agent(enablers = agentEnablers)

        every {
            enablerRepository.findAllById(
                match<Iterable<Long>> { ids ->
                    ids.toSet() == setOf(1L, 999L)
                }
            )
        } returns listOf(foundEnabler)

        val exception = assertThrows<AiBadRequestException> {
            agentEnablerUpdater.updateEnablers(
                agent = agent,
                rawEnablersValue = listOf(1, 999),
            )
        }

        assertEquals(ENABLER_NOT_FOUND, exception.errorCode)
        assertEquals(
            "Значение 999 не найдено в справочнике",
            exception.message,
        )

        /*
         * Важно: текущие связи не должны быть очищены,
         * потому что ошибка произошла до agent.enablers.clear().
         */
        assertEquals(setOf(existingAgentEnabler), agentEnablers)
    }

    @Test
    fun `updateEnablers should accept different number types`() {
        val enabler1 = enabler(id = 1L)
        val enabler2 = enabler(id = 2L)
        val enabler3 = enabler(id = 3L)

        val agentEnablers = mutableSetOf<EnablerEntity>()
        val agent = agent(enablers = agentEnablers)

        every {
            enablerRepository.findAllById(
                match<Iterable<Long>> { ids ->
                    ids.toSet() == setOf(1L, 2L, 3L)
                }
            )
        } returns listOf(enabler1, enabler2, enabler3)

        agentEnablerUpdater.updateEnablers(
            agent = agent,
            rawEnablersValue = listOf(
                1,
                2L,
                3.toShort(),
            ),
        )

        assertEquals(setOf(enabler1, enabler2, enabler3), agentEnablers)
    }

    private fun agent(
        enablers: MutableSet<EnablerEntity>,
    ): AIAgentEntity {
        return mockk<AIAgentEntity>(relaxed = true) {
            every {
                this@mockk.enablers
            } returns enablers
        }
    }

    private fun enabler(
        id: Long,
    ): EnablerEntity {
        return mockk<EnablerEntity>(relaxed = true) {
            every {
                this@mockk.id
            } returns id
        }
    }
}
  
```
