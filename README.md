```java
@Test
fun `updateQualityGate should update state without jira change when linked quality gate jira issue not found`() {
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

    val currentUserId = 1L

    val currentUser =
        UserDto(
            id = currentUserId,
            roles = setOf("PROJECT_OFFICE"),
            email = null,
            login = null,
            firstName = null,
            lastName = null,
            patronymic = null,
            phoneNumber = null,
            position = null,
            sberbankEmployee = null,
            companyId = null,
        )

    every {
        aiAgentRepository.findByIdOrNull(id = 1L)
    } returns aiAgent

    every {
        jiraIssueRepository.findByAgentIdAndTypeAndCode(
            agentId = 1L,
            code = "GATE1",
            qualityGateType = QualityGateType.quality_gate,
        )
    } returns listOf()

    every {
        agentQualityGateService.updateState(
            qualityGate = agentQualityGate,
            state = QualityGateState.checked,
        )
    } just Runs

    every {
        userInfoProvider.currentUser()
    } returns currentUser

    every {
        aiAgentRepository.saveAndFlush(
            entity = aiAgent,
        )
    } returns aiAgent

    // When
    service.updateQualityGate(
        id = 1L,
        request = UpdateAiAgentQualityGateRequest(
            qualityGateCode = "GATE1",
            state = QualityGateState.checked,
        ),
    )

    // Then
    verify(exactly = 1) {
        jiraIssueRepository.findByAgentIdAndTypeAndCode(
            agentId = 1L,
            code = "GATE1",
            qualityGateType = QualityGateType.quality_gate,
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

    verify(exactly = 1) {
        agentQualityGateService.updateState(
            qualityGate = agentQualityGate,
            state = QualityGateState.checked,
        )
    }

    verify(exactly = 1) {
        aiAgentRepository.saveAndFlush(
            entity = aiAgent,
        )
    }

    assertEquals("blocked", aiAgent.importStatus)
    assertEquals(currentUserId, aiAgent.updatedBy)
    assertNull(aiAgent.jiraFromStatus)
    assertNull(aiAgent.jiraFromUpdated)
    assertNull(aiAgent.jiraErrorCount)
}
```
