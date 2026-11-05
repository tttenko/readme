```java
/**
 * Обновляет разрешённые основные поля инициативы.
 *
 * Описание, контакты, финансовые эффекты, текущий бизнес-статус
 * и связанные сущности на этом этапе не изменяются.
 */
@Service
class JiraUpdatedInitiativePersistenceService(
    private val agentRepository: AIAgentRepository,
    private val organizationResolver: JiraInitiativeOrganizationResolver,
    private val initiativeTypeResolver: JiraInitiativeTypeResolver,
) {

    companion object {
        private const val MAX_AGENT_NAME_LENGTH = 255
        private val log by logger()
    }

    /**
     * Под блокировкой строки проверяет дату изменения и сохраняет поля ai_agent.
     *
     * Для инициатив, затронутых Task в текущем запуске, сравнивает дату Jira
     * со значением updated, которое было до обработки Task.
     * Незавершённую синхронизацию разрешает повторить после сбоя.
     */
    @Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = [Exception::class])
    fun updateInitiative(
        agentId: Long,
        issue: SearchIssueDto,
        jiraUpdated: LocalDateTime,
        referenceData: JiraImportReferenceData,
        pultUpdatedBeforeTaskSync: Map<Long, LocalDateTime?>,
    ): JiraUpdatedInitiativeResult {
        val agent = agentRepository.findByIdForUpdate(agentId) ?: return JiraUpdatedInitiativeResult.SKIPPED
        if (agent.disabled == true) return JiraUpdatedInitiativeResult.SKIPPED

        val pultUpdated = if (pultUpdatedBeforeTaskSync.containsKey(agentId)) {
            pultUpdatedBeforeTaskSync[agentId]
        } else {
            agent.updated
        }

        val synchronizationIsIncomplete = agent.jiraFromStatus == "inProgress" || agent.jiraFromStatus == "error"

        if (!synchronizationIsIncomplete && pultUpdated != null && pultUpdated.isAfter(jiraUpdated)) {
            log.debug("Skipping initiative updated later in Pult: agentId={}, jiraKey={}", agentId, issue.key)
            return JiraUpdatedInitiativeResult.SKIPPED
        }

        val fields = issue.fields ?: throw AiBadRequestException(
            errorCode = JIRA_SYNC_ERROR,
            message = "Jira initiative fields are missing: ${issue.key}"
        )

        val summary = fields.summary?.takeIf(String::isNotBlank) ?: throw AiBadRequestException(
            errorCode = JIRA_SYNC_ERROR,
            message = "Jira initiative summary is missing: ${issue.key}"
        )

        val organization = organizationResolver.resolveOrganization(
            initiatorUnits = fields.customfield_30000,
            executorUnits = fields.customfield_30001,
            referenceData = referenceData
        )

        if (organization.block != null || organization.division != null) {
            agent.block = organization.block
            agent.division = organization.division
        } else {
            log.warn(
                "Organization is not resolved from Jira; existing Pult values retained: jiraKey={}, agentId={}, block={}, division={}",
                issue.key, agentId, agent.block?.code, agent.division?.code
            )
        }

        agent.agentName = summary.take(MAX_AGENT_NAME_LENGTH)
        agent.initiativeType = initiativeTypeResolver.resolveInitiativeType(
            labels = fields.labels,
            initiativeTypesByCode = referenceData.initiativeTypesByCode
        )
        agent.jiraFromStatus = "inProgress"
        agent.updated = LocalDateTime.now()

        agentRepository.save(agent)

        log.debug("Updated initiative fields from Jira: agentId={}, jiraKey={}", agentId, issue.key)
        return JiraUpdatedInitiativeResult.UPDATED
    }
}
```
