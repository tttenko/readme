```java

/**
 * Обновляет разрешённые основные поля инициативы и финансовые эффекты.
 *
 * На переходный период финансовые эффекты сохраняются одновременно
 * в legacy-поля ai_agent и в initiative_financial_effect.
 *
 * Описание, контакты и текущий бизнес-статус на этом этапе не изменяются.
 */
@Service
class JiraUpdatedInitiativePersistenceService(
    private val agentRepository: AIAgentRepository,
    private val organizationResolver: JiraInitiativeOrganizationResolver,
    private val initiativeTypeResolver: JiraInitiativeTypeResolver,
    private val numericValueParser: JiraNumericValueParser,
    private val financialEffectSynchronizer: JiraInitiativeFinancialEffectSynchronizer,
) {

    companion object {
        private const val MAX_AGENT_NAME_LENGTH = 255
        private val log by logger()
    }

    /**
     * Под блокировкой строки проверяет дату изменения и сохраняет поля инициативы.
     * Финансовые эффекты синхронизируются в той же транзакции.
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

        // TODO: временный legacy-маппинг.
        // Удалить agentEffectOptimization и agentEffectRevenue
        // после полного перехода на initiative_financial_effect.
        agent.agentEffectOptimization = parseEffect(issue.key, "customfield_34300", fields.customfield_34300)
        agent.agentEffectRevenue = parseEffect(issue.key, "customfield_30401", fields.customfield_30401)

        agent.jiraFromStatus = "inProgress"
        agent.updated = LocalDateTime.now()

        agentRepository.save(agent)

        financialEffectSynchronizer.synchronizeFinancialEffects(
            initiative = agent,
            jiraKey = issue.key,
            revenueValue = fields.customfield_30401,
            optimizationValue = fields.customfield_34300
        )

        log.debug("Updated initiative fields from Jira: agentId={}, jiraKey={}", agentId, issue.key)
        return JiraUpdatedInitiativeResult.UPDATED
    }

    /** Извлекает числовой эффект для временного legacy-маппинга общим парсером FR1/FR2. */
    private fun parseEffect(jiraKey: String?, fieldName: String, value: String?): BigDecimal? {
        if (value.isNullOrBlank()) return null

        val effect = numericValueParser.parseFirst(value)
        if (effect == null) {
            log.warn("Cannot parse Jira initiative effect: jiraKey={}, field={}, value={}", jiraKey, fieldName, value)
        }
        return effect
    }
}
```
