```java
@Test
fun `updateQualityGate should throw bad request when more than one linked quality gate jira issue exists`() {
    // Given
    val qualityGate =
        QualityGateEntity(
            code = "GATE1",
        )

    val agentQualityGate =
        AIAgentQualityGateEntity().also {
            it.qualityGate = qualityGate
        }

    val aiAgent =
        AIAgentEntity().also {
            it.id = 1L
            it.qualityGates = mutableSetOf(agentQualityGate)
        }

    val firstLinkedQualityGateJiraIssue =
        JiraIssueEntity().also {
            it.jiraKey = "TEST-1"
        }

    val secondLinkedQualityGateJiraIssue =
        JiraIssueEntity().also {
            it.jiraKey = "TEST-2"
        }

    every {
        aiAgentRepository.findByIdOrNull(1L)
    } returns aiAgent

    every {
        jiraIssueRepository.findByAgentIdAndTypeAndCode(
            agentId = 1L,
            code = "GATE1",
            qualityGateType = QualityGateType.quality_gate,
        )
    } returns listOf(
        firstLinkedQualityGateJiraIssue,
        secondLinkedQualityGateJiraIssue,
    )

    every {
        messageProvider[Metadata.ErrorMessages.MORE_THAN_ONE_JIRA_ISSUE]
    } returns "Не удалось синхронизировать статус с JIRA. Найдено несколько тикетов в JIRA"

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
        Metadata.ErrorMessages.MORE_THAN_ONE_JIRA_ISSUE,
        exception.errorCode,
    )

    verify(exactly = 1) {
        jiraIssueRepository.findByAgentIdAndTypeAndCode(
            agentId = 1L,
            code = "GATE1",
            qualityGateType = QualityGateType.quality_gate,
        )
    }

    verify(exactly = 0) {
        jiraChangeCreator.createQualityGateChange(
            any(),
            any(),
            any(),
            any(),
        )
    }

    verify(exactly = 0) {
        agentQualityGateService.updateState(
            any(),
            any(),
        )
    }

    verify(exactly = 0) {
        aiAgentRepository.saveAndFlush(
            any<AIAgentEntity>(),
        )
    }
}
```
