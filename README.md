```java
@Test
fun `update should parse planned date when value is local date time string`() {
    val agent = agent(id = 1L)

    val status = StatusEntity().also {
        it.id = 1L
        it.code = "pilot"
    }

    val request = listOf(
        mapOf(
            "status" to "pilot",
            "plannedDate" to "2026-06-01T00:00:00.000",
        )
    )

    every {
        agentStatusSlaRepository.findAllByAiAgentId(
            agentId = 1L,
        )
    } returns emptyList()

    every {
        statusRepository.findAllByDisabledIsFalse()
    } returns listOf(status)

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
    assertEquals(LocalDate.of(2026, 6, 1), result.first().plannedDate)

    verify {
        agentStatusSlaRepository.saveAll(
            entities = any<List<AgentStatusSlaEntity>>(),
        )
    }
}

@Test
fun `update should parse planned date when value is local date time`() {
    val agent = agent(id = 1L)

    val status = StatusEntity().also {
        it.id = 1L
        it.code = "pilot"
    }

    val request = listOf(
        mapOf(
            "status" to "pilot",
            "plannedDate" to LocalDateTime.of(2026, 6, 1, 15, 30),
        )
    )

    every {
        agentStatusSlaRepository.findAllByAiAgentId(
            agentId = 1L,
        )
    } returns emptyList()

    every {
        statusRepository.findAllByDisabledIsFalse()
    } returns listOf(status)

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
    assertEquals(LocalDate.of(2026, 6, 1), result.first().plannedDate)
}

@Test
fun `update should throw exception when planned date has unsupported type`() {
    val agent = agent(id = 1L)

    val request = listOf(
        mapOf(
            "status" to "pilot",
            "plannedDate" to 123,
        )
    )

    every {
        messageProvider[WRONG_STATUS_SLA_PLANNED_DATE_VALUE]
    } returns "Wrong statusSla plannedDate value"

    val exception = assertThrows<AiBadRequestException> {
        statusSlaUpdater.update(
            agent = agent,
            rawStatusSlaValue = request,
        )
    }

    assertEquals(
        WRONG_STATUS_SLA_PLANNED_DATE_VALUE,
        exception.errorCode,
    )
}
```
