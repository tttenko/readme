```java
assertThat(session.statusSlaByInitiativeId)
        .containsOnlyKeys(1L)

    assertThat(session.statusSlaByInitiativeId.getValue(1L))
        .containsExactly(sla)

    verify(exactly = 1) {
        agentStatusSlaRepository.findAllByAiAgentIdIn(initiativeIds)
    }
  ```
