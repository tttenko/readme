```java
Test
fun `updateQualityGate should throw bad request when quality gate not found`() {
    // Given
    val aiAgent =
        AIAgentEntity().also {
            it.id = 1L
        }

    every {
        aiAgentRepository.findByIdOrNull(id = 1L)
    } returns aiAgent

    every {
        messageProvider[Metadata.ErrorMessages.BAD_REQUEST]
    } returns "bad request"

    // When
    val exception =
        assertThrows<AiBadRequestException> {
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
        Metadata.ErrorMessages.BAD_REQUEST,
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
        jiraChangeCreator.createQualityGateChange(
            aiAgent = any(),
            qualityGateCode = any(),
            qualityGateState = any(),
            linkedQualityGateJiraIssue = any(),
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
