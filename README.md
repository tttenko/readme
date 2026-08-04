```java
@Test
fun `statusSlaByInitiativeId should group SLA records by initiative id`() {
    val firstSla = createStatusSla().apply {
        primaryKey.aiAgentId = 1L
    }

    val secondSla = createStatusSla().apply {
        primaryKey.aiAgentId = 1L
    }

    val thirdSla = createStatusSla().apply {
        primaryKey.aiAgentId = 2L
    }

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
  ```
