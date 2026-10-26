```java

/**
 * Создаёт новую инициативу Пульта на основании данных Jira.
 *
 * В рамках одной транзакции текущая реализация FR1:
 * 1. создаёт AIAgentEntity;
 * 2. создаёт финансовые эффекты;
 * 3. создаёт связи со стратегиями;
 * 4. сохраняет задействованные ресурсы;
 * 5. создаёт developer/customer контакты;
 * 6. создаёт связи с энейблерами;
 * 7. инициализирует quality gates в состоянии unchecked;
 * 8. создаёт связь с основной CROSSGOAL initiative;
 * 9. создаёт связи со всеми найденными GIGAUSAGE issues.
 *
 * Monitoring epic, связанные задачи и последующий расчёт статуса
 * будут обрабатываться на следующих этапах FR1.
 *
 * Каждая Jira-инициатива обрабатывается в отдельной транзакции,
 * поэтому ошибка одной инициативы не откатывает обработку остальных.
 */
@Service
class JiraNewInitiativeCreationService(
    private val agentRepository: AIAgentRepository,
    private val agentStrategyRepository: AgentStrategyRepository,
    private val involvedResourceRepository: InvolvedResourceRepository,

    private val organizationResolver: JiraInitiativeOrganizationResolver,
    private val initiativeTypeResolver: JiraInitiativeTypeResolver,
    private val numericValueParser: JiraNumericValueParser,
    private val strategyResolver: JiraStrategyResolver,
    private val involvedResourceResolver: JiraInvolvedResourceResolver,

    private val contactCreator: JiraInitiativeContactCreator,
    private val enablerCreator: JiraInitiativeEnablerCreator,
    private val qualityGateCreator: JiraInitiativeQualityGateCreator,
    private val issueRelationCreator: JiraInitiativeIssueRelationCreator,
    private val financialEffectSynchronizer: JiraInitiativeFinancialEffectSynchronizer,
) {

    companion object {
        private const val ANALYSIS_STATUS_CODE = "analysis"

        private const val IMPORT_STATUS_BLOCKED = "blocked"
        private const val JIRA_FROM_STATUS_IN_PROGRESS = "inProgress"
        private const val JIRA_LINK_DONE = "done"

        private const val MAX_AGENT_NAME_LENGTH = 255
        private const val MAX_AGENT_DESCRIPTION_LENGTH = 1000

        private val log by logger()
    }

    /**
     * Создаёт новую инициативу Пульта на основании данных Jira.
     *
     * В рамках одной транзакции:
     * 1. создаёт AIAgentEntity;
     * 2. создаёт финансовые эффекты;
     * 3. создаёт связи со стратегиями;
     * 4. сохраняет involved_resource;
     * 5. создаёт developer/customer контакты;
     * 6. создаёт связи с энейблерами;
     * 7. инициализирует quality gates;
     * 8. создаёт связь с основной CROSSGOAL initiative;
     * 9. создаёт связи с GIGAUSAGE issues.
     *
     * Если определить block/division не удалось,
     * инициатива не создаётся.
     *
     * Monitoring epic и связанные Task обрабатываются
     * отдельным этапом после завершения текущей транзакции.
     *
     * @param issue исходная Jira-инициатива
     * @param referenceData справочники текущего запуска scheduler
     * @return id созданной инициативы или null, если инициатива была пропущена
     */
    @Transactional(
        propagation = Propagation.REQUIRES_NEW,
        rollbackFor = [Exception::class],
    )
    fun createInitiativeFromJira(
        issue: SearchIssueDto,
        referenceData: JiraImportReferenceData,
    ): Long? {
        val jiraKey = requireNotNull(issue.key) {
            "Jira initiative key must not be null"
        }

        log.debug(
            "Started creating new Jira initiative in Pult: jiraKey={}",
            jiraKey,
        )

        val analysisStatus = requireNotNull(
            referenceData.statusesByCode[ANALYSIS_STATUS_CODE]
        ) {
            "Active status '$ANALYSIS_STATUS_CODE' is not configured"
        }

        val organization = organizationResolver.resolveOrganization(
            initiatorUnits = issue.fields?.customfield_30000,
            executorUnits = issue.fields?.customfield_30001,
            referenceData = referenceData,
        )

        if (organization.block == null && organization.division == null) {
            log.warn(
                "Skipping new Jira initiative because block and division were not resolved: " +
                    "jiraKey={}, customfield_30000={}, customfield_30001={}",
                jiraKey,
                issue.fields?.customfield_30000,
                issue.fields?.customfield_30001,
            )

            return null
        }

        val initiativeType = initiativeTypeResolver.resolveInitiativeType(
            labels = issue.fields?.labels,
            initiativeTypesByCode = referenceData.initiativeTypesByCode,
        )

        val optimizationEffect = parseEffect(
            jiraKey = jiraKey,
            jiraFieldName = "customfield_34300",
            jiraFieldValue = issue.fields?.customfield_34300,
        )

        val revenueEffect = parseEffect(
            jiraKey = jiraKey,
            jiraFieldName = "customfield_30401",
            jiraFieldValue = issue.fields?.customfield_30401,
        )

        val currentDateTime = LocalDateTime.now()

        val agent = AIAgentEntity(
            agentId = jiraKey,
            agentName = truncate(
                jiraKey = jiraKey,
                jiraFieldName = "summary",
                value = issue.fields?.summary,
                maxLength = MAX_AGENT_NAME_LENGTH,
            ),
            agentDescription = truncate(
                jiraKey = jiraKey,
                jiraFieldName = "description",
                value = issue.fields?.description,
                maxLength = MAX_AGENT_DESCRIPTION_LENGTH,
            ),
            agentJiraUrl = jiraKey,

            // TODO: временный legacy-маппинг.
            // Удалить agentEffectOptimization и agentEffectRevenue
            // после полного перехода на initiative_financial_effect.
            agentEffectOptimization = optimizationEffect,
            agentEffectRevenue = revenueEffect,

            initiativeType = initiativeType,
            block = organization.block,
            division = organization.division,
            agentStatus = analysisStatus,
            importStatus = IMPORT_STATUS_BLOCKED,
            jiraFromStatus = JIRA_FROM_STATUS_IN_PROGRESS,
        ).apply {
            created = currentDateTime
        }

        val savedAgent = agentRepository.save(agent)

        log.debug(
            "Created ai_agent from Jira: jiraKey={}, agentId={}, block={}, division={}, initiativeType={}",
            jiraKey,
            savedAgent.id,
            savedAgent.block?.code,
            savedAgent.division?.code,
            savedAgent.initiativeType?.code,
        )

        /*
         * Новое хранение финансовых эффектов.
         *
         * Пока выполняется параллельно с legacy-полями:
         * ai_agent.agent_effect_optimization
         * ai_agent.agent_effect_revenue
         */
        financialEffectSynchronizer.createFinancialEffects(
            initiative = savedAgent,
            jiraKey = jiraKey,
            revenueValue = issue.fields?.customfield_30401,
            optimizationValue = issue.fields?.customfield_34300,
        )

        createStrategies(
            agent = savedAgent,
            issue = issue,
            referenceData = referenceData,
        )

        createInvolvedResources(
            agent = savedAgent,
            issue = issue,
            jiraKey = jiraKey,
            currentDateTime = currentDateTime,
        )

        contactCreator.createContacts(
            agent = savedAgent,
            issue = issue,
        )

        enablerCreator.createEnablers(
            agent = savedAgent,
            issue = issue,
            referenceData = referenceData,
        )

        qualityGateCreator.createQualityGates(
            agent = savedAgent,
            referenceData = referenceData,
            currentDateTime = currentDateTime,
        )

        issueRelationCreator.createIssueRelations(
            agent = savedAgent,
            issue = issue,
            currentDateTime = currentDateTime,
        )

        log.info(
            "Successfully created new Jira initiative in Pult: jiraKey={}, agentId={}",
            jiraKey,
            savedAgent.id,
        )

        return savedAgent.id
    }

    /**
     * Создаёт связи новой инициативы со стратегиями.
     *
     * Для найденных Jira-связей создаётся agent_strategy
     * со значением jira_link=done.
     */
    private fun createStrategies(
        agent: AIAgentEntity,
        issue: SearchIssueDto,
        referenceData: JiraImportReferenceData,
    ) {
        val strategies = strategyResolver.resolveStrategies(
            issue = issue,
            strategiesByJiraKey = referenceData.strategiesByJiraKey,
        )

        if (strategies.isEmpty()) {
            log.debug(
                "No strategies found for Jira initiative: jiraKey={}",
                issue.key,
            )
            return
        }

        val agentStrategies = strategies.map { strategy ->
            AgentStrategyEntity(
                agent = agent,
                strategy = strategy,
                jiraLink = JIRA_LINK_DONE,
            )
        }

        agentStrategyRepository.saveAll(agentStrategies)

        log.debug(
            "Created agent strategies: jiraKey={}, agentId={}, strategyIds={}",
            issue.key,
            agent.id,
            strategies.map { strategy -> strategy.id },
        )
    }

    /**
     * Создаёт involved_resource для новой инициативы.
     *
     * Каждое успешно распарсенное Jira-поле ресурсов
     * сохраняется отдельной записью.
     */
    private fun createInvolvedResources(
        agent: AIAgentEntity,
        issue: SearchIssueDto,
        jiraKey: String,
        currentDateTime: LocalDateTime,
    ) {
        val resources = involvedResourceResolver.resolveInvolvedResources(
            jiraKey = jiraKey,
            issue = issue,
        )

        if (resources.isEmpty()) {
            log.debug(
                "No involved resources found for Jira initiative: jiraKey={}",
                jiraKey,
            )
            return
        }

        val involvedResourceEntities = resources.map { resource ->
            InvolvedResourceEntity().apply {
                id = InvolvedResourceEmbeddedId(
                    aiAgentId = agent.id,
                    source = resource.source,
                    type = resource.type,
                )
                value = resource.value
                timeAllocated = null
                created = currentDateTime
                updated = currentDateTime
                aiAgent = agent
            }
        }

        involvedResourceRepository.saveAll(involvedResourceEntities)

        log.debug(
            "Created involved resources: jiraKey={}, agentId={}, resources={}",
            jiraKey,
            agent.id,
            resources.map { resource ->
                "${resource.source}/${resource.type}=${resource.value}"
            },
        )
    }

    /**
     * Извлекает числовое значение эффекта из Jira-поля.
     *
     * TODO: удалить метод после удаления legacy-полей
     * agentEffectOptimization и agentEffectRevenue из FR1-маппинга.
     *
     * Некорректное непустое значение не блокирует создание инициативы,
     * но фиксируется в логах.
     */
    private fun parseEffect(
        jiraKey: String,
        jiraFieldName: String,
        jiraFieldValue: String?,
    ): BigDecimal? {
        if (jiraFieldValue.isNullOrBlank()) {
            return null
        }

        val numericValue = numericValueParser.parseFirst(jiraFieldValue)

        if (numericValue == null) {
            log.warn(
                "Cannot parse initiative effect from Jira: jiraKey={}, field={}, value={}",
                jiraKey,
                jiraFieldName,
                jiraFieldValue,
            )
        }

        return numericValue
    }

    /**
     * Ограничивает строковое значение максимально допустимой длиной.
     *
     * Если значение превышает допустимый размер,
     * оно обрезается и операция фиксируется в DEBUG-логе.
     */
    private fun truncate(
        jiraKey: String,
        jiraFieldName: String,
        value: String?,
        maxLength: Int,
    ): String? {
        if (value == null || value.length <= maxLength) {
            return value
        }

        log.debug(
            "Jira field was truncated while creating initiative: jiraKey={}, field={}, originalLength={}, maxLength={}",
            jiraKey,
            jiraFieldName,
            value.length,
            maxLength,
        )

        return value.take(maxLength)
    }
}

```
