```java
@Test
fun `updateQualityGate should throw not found when agent not found`() {
    // Given
    every {
        aiAgentRepository.findByIdOrNull(id = 1L)
    } returns null

    every {
        messageProvider[Metadata.ErrorMessages.AGENT_NOT_FOUND]
    } returns "agent not found"

    // When
    val exception =
        assertThrows<AiNotFoundException> {
            service.updateQualityGate(
                id = 1L,
                request = UpdateAiAgentQualityGateRequest(
                    qualityGateCode = "GATE1",
                    state = QualityGateState.checked,
                ),
            )
        }

    // Then
    assertEquals(
        Metadata.ErrorMessages.AGENT_NOT_FOUND,
        exception.errorCode,
    )

    verify(exactly = 0) {
        jiraIssueRepository.findByAgentIdAndTypeAndCode(
            agentId = any(),
            code = any(),
            qualityGateType = any(),
        )
    }

    verify(exactly = 0) {
        agentQualityGateService.updateState(
            qualityGate = any(),
            state = any(),
        )
    }

    verify(exactly = 0) {
        aiAgentRepository.saveAndFlush(
            entity = any<AIAgentEntity>(),
        )
    }
}
```
