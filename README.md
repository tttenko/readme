```java
@Transactional
open fun updateQualityGate(
    id: Long,
    request: UpdateAiAgentQualityGateRequest,
) {
    val aiAgent = aiAgentRepository.findByIdOrNull(id)
        ?: throw AiNotFoundException(
            errorCode = Metadata.ErrorMessages.AGENT_NOT_FOUND,
            message = messageProvider[Metadata.ErrorMessages.AGENT_NOT_FOUND],
        )

    val requestedQualityGateCode = request.qualityGateCode
    val requestedQualityGateState = request.state

    val agentQualityGateForUpdate =
        aiAgent.qualityGates.find { agentQualityGate ->
            agentQualityGate.qualityGate?.code == requestedQualityGateCode
        } ?: throw AiBadRequestException(
            message = messageProvider[Metadata.ErrorMessages.BAD_REQUEST],
            errorCode = Metadata.ErrorMessages.BAD_REQUEST,
        )

    val hasJiraChangeBeenCreated =
        createJiraChangeIfSingleLinkedQualityGateIssueExists(
            aiAgent = aiAgent,
            qualityGateCode = requestedQualityGateCode,
            qualityGateState = requestedQualityGateState,
        )

    agentQualityGateService.updateQualityGateState(
        agentQualityGate = agentQualityGateForUpdate,
        qualityGateState = requestedQualityGateState,
    )

    lockAiAgentImportAfterQualityGateUpdate(
        aiAgent = aiAgent,
        hasJiraChangeBeenCreated = hasJiraChangeBeenCreated,
    )
}

private fun createJiraChangeIfSingleLinkedQualityGateIssueExists(
    aiAgent: AIAgentEntity,
    qualityGateCode: String,
    qualityGateState: QualityGateState,
): Boolean {
    val linkedQualityGateJiraIssues =
        jiraIssueRepository.findQualityGateJiraIssues(
            agentId = aiAgent.id,
            qualityGateCode = qualityGateCode,
        )

    return when (linkedQualityGateJiraIssues.size) {

        0 -> false

        1 -> {
            qualityGateJiraChangeCreator.createQualityGateChange(
                aiAgent = aiAgent,
                qualityGateCode = qualityGateCode,
                qualityGateState = qualityGateState,
                linkedQualityGateJiraIssue = linkedQualityGateJiraIssues.first(),
            )

            true
        }

        else -> throw AiBadRequestException(
            message = messageProvider[Metadata.ErrorMessages.MORE_THAN_ONE_JIRA_ISSUE],
            errorCode = Metadata.ErrorMessages.MORE_THAN_ONE_JIRA_ISSUE,
        )
    }
}

open fun createQualityGateChange(
        aiAgent: AIAgentEntity,
        qualityGateCode: String,
        qualityGateState: QualityGateState,
        linkedQualityGateJiraIssue: JiraIssueEntity,
    ) {
        val qualityGateChangePayload =
            objectMapper.valueToTree<JsonNode>(
                QualityGateJiraChangePayload(
                    qualityGateCode = qualityGateCode,
                    status = qualityGateState.name,
                    jiraKey = linkedQualityGateJiraIssue.jiraKey,
                )
            )

        val qualityGateJiraChange =
            JiraChangeEntity(
                agent = aiAgent,
                changeType = QUALITY_GATE_CHANGE_TYPE,
                payload = qualityGateChangePayload,
                created = LocalDateTime.now(),
            )

        jiraChangeRepository.save(
            entity = qualityGateJiraChange,
        )
    }

    private data class QualityGateJiraChangePayload(
        val qualityGateCode: String,
        val status: String,
        val jiraKey: String?,
    )

    private companion object {

        private const val QUALITY_GATE_CHANGE_TYPE = "quality_gate"
    }

private fun lockAiAgentImportAfterQualityGateUpdate(
    aiAgent: AIAgentEntity,
    hasJiraChangeBeenCreated: Boolean,
) {
    aiAgent.importStatus = IMPORT_STATUS_BLOCKED
    aiAgent.updatedBy = userInfoProvider.currentUser().id

    if (hasJiraChangeBeenCreated) {
        aiAgent.jiraFromStatus = JIRA_FROM_STATUS_WAITING
        aiAgent.jiraFromUpdated = LocalDateTime.now()
        aiAgent.jiraErrorCount = 0
    }

    aiAgentRepository.saveAndFlush(
        entity = aiAgent,
    )
}


```
