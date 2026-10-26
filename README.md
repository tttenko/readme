```java

private fun processAgent(
    agent: AIAgentEntity,
    jiraResponse: IssueDto,
    monitoringTasks: List<SearchIssueDto>?,
) {
    val strategies = strategyService.findAll()
    val qualityGates = qualityGateRepository.findAllByDisabledIsFalse()
    val enabledDivisions = divisionRepository.findAll()
    val enabledBlocks = blockRepository.findAllByDisabledIsFalse()
    val initiativeTypes = initiativeTypeRepository.findAll()

    val enablersByNormalizedName = enablerRepository.findAll()
        .asSequence()
        .filter { it.disabled != true }
        .mapNotNull { enabler ->
            enablerNameNormalizer.normalize(enabler.name)?.let { normalizedName ->
                normalizedName to enabler
            }
        }
        .toMap()

    createInitiativeJiraIssue(
        agent = agent,
        jiraResponse = jiraResponse,
    )

    agent.apply {
        jiraResponse.fields?.summary?.let { summary ->
            agentName = summary.substring(
                startIndex = 0,
                endIndex = minOf(255, summary.length),
            )
        }

        val (resolvedDivision, resolvedBlock) =
            determineDivisionAndBlock(
                customfield_30000 = jiraResponse.fields?.customfield_30000,
                customfield_30001 = jiraResponse.fields?.customfield_30001,
                customfield_30002 = jiraResponse.fields?.customfield_30002,
                enabledDivisions = enabledDivisions,
                enabledBlocks = enabledBlocks,
            )

        division = resolvedDivision
        block = resolvedBlock

        initiativeType = determineInitiativeType(
            labels = jiraResponse.fields?.labels,
            initiativeTypes = initiativeTypes,
        )

        // TODO: временный legacy-маппинг.
        // Удалить agentEffectOptimization и agentEffectRevenue
        // после полного перехода на initiative_financial_effect.
        agentEffectOptimization = jiraNumericValueParser.parseFirst(
            jiraResponse.fields?.customfield_34300
        )

        agentEffectRevenue = jiraNumericValueParser.parseFirst(
            jiraResponse.fields?.customfield_30401
        )

        jiraFromStatus = "inProgress"
        updated = LocalDateTime.now()
    }

    agentRepository.save(agent)

    /*
     * Новое хранение финансовых эффектов.
     *
     * Пока выполняется параллельно с legacy-полями:
     * ai_agent.agent_effect_optimization
     * ai_agent.agent_effect_revenue
     */
    financialEffectSynchronizer.synchronizeFinancialEffects(
        initiative = agent,
        jiraKey = jiraResponse.key,
        revenueValue = jiraResponse.fields?.customfield_30401,
        optimizationValue = jiraResponse.fields?.customfield_34300,
    )

    updateAgentStrategies(
        agent = agent,
        strategies = strategies,
        jiraResponse = jiraResponse,
    )

    updateInvolvedResources(
        agent = agent,
        jiraResponse = jiraResponse,
    )

    updateEnablers(
        agent = agent,
        jiraResponse = jiraResponse,
        enablersByNormalizedName = enablersByNormalizedName,
    )

    updateGigausage(
        agent = agent,
        jiraResponse = jiraResponse,
    )

    syncWithMonitoringEpic(
        agent = agent,
        jiraResponse = jiraResponse,
        qualityGates = qualityGates,
        monitoringTasks = monitoringTasks,
    )

    log.debug(
        "В Пульт синхронизирована с Jira инициатива с ключом ${jiraResponse.key}"
    )
}

```
