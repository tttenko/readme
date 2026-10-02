```java

/**
 * Обновляет разрешённые основные поля инициативы.
 *
 * Описание, контакты, текущий бизнес-статус и связанные сущности
 * в этом классе не изменяются.
 */
@Service
class JiraUpdatedInitiativePersistenceService(
    private val agentRepository: AIAgentRepository,
    private val organizationResolver: JiraInitiativeOrganizationResolver,
    private val initiativeTypeResolver: JiraInitiativeTypeResolver,
    private val numericValueParser: JiraNumericValueParser,
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
        agent.agentEffectOptimization = parseEffect(issue.key, "customfield_34300", fields.customfield_34300)
        agent.agentEffectRevenue = parseEffect(issue.key, "customfield_30401", fields.customfield_30401)
        agent.jiraFromStatus = "inProgress"
        agent.updated = LocalDateTime.now()

        agentRepository.save(agent)
        log.debug("Updated initiative fields from Jira: agentId={}, jiraKey={}", agentId, issue.key)
        return JiraUpdatedInitiativeResult.UPDATED
    }

    /** Извлекает числовой эффект общим парсером FR1/FR2. */
    private fun parseEffect(jiraKey: String?, fieldName: String, value: String?): BigDecimal? {
        if (value.isNullOrBlank()) return null

        val effect = numericValueParser.parseFirst(value)
        if (effect == null) {
            log.warn("Cannot parse Jira initiative effect: jiraKey={}, field={}, value={}", jiraKey, fieldName, value)
        }
        return effect
    }
}

* Для поиска, сопоставления и сохранения использует существующие
 * компоненты FR1. Jira Search выполняется вне транзакций сохранения.
 */
@Service
class JiraUpdatedInitiativeMonitoringService(
    private val monitoringEpicResolver: JiraMonitoringEpicResolver,
    private val monitoringTaskSearchService: JiraMonitoringTaskSearchService,
    private val taskQualityGateMatcher: JiraTaskQualityGateMatcher,
    private val monitoringPersistenceService: JiraMonitoringPersistenceService,
    private val monitoringTaskCleanupService: JiraUpdatedMonitoringTaskCleanupService,
    private val completionService: JiraUpdatedInitiativeCompletionService,
) {

    private val log by logger()

    /**
     * Получает полный набор Task мониторинга и сохраняет результат.
     *
     * Для AI-эффективности и инициатив без monitoring Epic завершает
     * синхронизацию без запроса Task. Epic сохраняется после успешного
     * поиска Task и проверки наличия хотя бы одного корректного этапа.
     */
    fun synchronizeMonitoring(
        agentId: Long,
        issue: SearchIssueDto,
        referenceData: JiraImportReferenceData,
        maxResults: Int,
        jiraErrorTracker: JiraErrorTracker,
    ) {
        val jiraKey = issue.key ?: throw AiBadRequestException(
            errorCode = JIRA_SYNC_ERROR,
            message = "Jira initiative key is missing: agentId=$agentId"
        )

        if (!monitoringEpicResolver.isMonitoringRequired(issue.fields?.labels)) {
            completionService.completeWithoutMonitoring(agentId, jiraKey, referenceData.statusesByCode)
            log.debug("Skipped monitoring for AI-effectiveness initiative: agentId={}, jiraKey={}", agentId, jiraKey)
            return
        }

        val monitoringEpic = monitoringEpicResolver.findMonitoringEpic(jiraKey, issue.fields?.issuelinks)
        if (monitoringEpic == null) {
            completionService.completeWithoutMonitoring(agentId, jiraKey, referenceData.statusesByCode)
            log.debug("Monitoring Epic was not found: agentId={}, jiraKey={}", agentId, jiraKey)
            return
        }

        val monitoringTasks = monitoringTaskSearchService.searchMonitoringTasks(
            epicKey = monitoringEpic.jiraKey,
            maxResults = maxResults,
            jiraErrorTracker = jiraErrorTracker
        )

        val taskMatches = taskQualityGateMatcher.matchTasks(
            initiativeJiraKey = jiraKey,
            tasks = monitoringTasks,
            qualityGates = referenceData.qualityGates
        )

        val hasValidStage = taskMatches.any { match ->
            match.qualityGate.type == QualityGateType.status &&
                match.qualityGate.status?.code != null &&
                !match.task.fields?.status?.id.isNullOrBlank()
        }

        if (!hasValidStage) {
            throw AiBadRequestException(
                errorCode = JIRA_SYNC_ERROR,
                message = "No valid monitoring stages for Jira initiative $jiraKey"
            )
        }

        val epicIssueId = monitoringPersistenceService.saveMonitoringEpic(agentId, monitoringEpic)

        monitoringPersistenceService.saveMonitoringData(
            agentId = agentId,
            initiativeJiraKey = jiraKey,
            monitoringEpicIssueId = epicIssueId,
            monitoringEpicKey = monitoringEpic.jiraKey,
            taskMatches = taskMatches,
            referenceData = referenceData
        )

        monitoringTaskCleanupService.deleteObsoleteTasks(agentId, epicIssueId, taskMatches)

        log.debug(
            "Updated initiative monitoring: agentId={}, jiraKey={}, epicKey={}, tasks={}, matchedTasks={}",
            agentId, jiraKey, monitoringEpic.jiraKey, monitoringTasks.size, taskMatches.size
        )
    }
}

```
