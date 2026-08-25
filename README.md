```java

/**
 * Создаёт новую инициативу Пульта на основании данных Jira.
 *
 * В рамках одной транзакции:
 * 1. создаёт AIAgentEntity;
 * 2. создаёт связи со стратегиями;
 * 3. сохраняет involved_resource;
 * 4. создаёт developer/customer контакты;
 * 5. создаёт связи с энейблерами;
 * 6. инициализирует quality gates;
 * 7. создаёт связь с основной CROSSGOAL initiative;
 * 8. создаёт связи с GIGAUSAGE issues.
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

    if (
        organization.block == null &&
        organization.division == null
    ) {
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

```
