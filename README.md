```java
@Test
fun `updateQualityGate should create jira change when single linked quality gate issue exists`() {
    // Given
    val qualityGate = QualityGateEntity(
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
            it.jiraStatus = "done"
        }

    val currentUser =
        UserDto(
            id = 1L,
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

    val linkedQualityGateJiraIssue =
        JiraIssueEntity().also {
            it.jiraKey = "TEST-1"
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
    } returns listOf(linkedQualityGateJiraIssue)

    every {
        jiraChangeCreator.createQualityGateChange(
            aiAgent = aiAgent,
            qualityGateCode = "GATE1",
            qualityGateState = QualityGateState.checked,
            linkedQualityGateJiraIssue = linkedQualityGateJiraIssue,
        )
    } just Runs

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

    verify(exactly = 1) {
        jiraChangeCreator.createQualityGateChange(
            aiAgent = aiAgent,
            qualityGateCode = "GATE1",
            qualityGateState = QualityGateState.checked,
            linkedQualityGateJiraIssue = linkedQualityGateJiraIssue,
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
}
```
