```java

fun getAIAgentList(
    showcaseRequestDto: ShowcaseRequestDto,
    hasDeviation: Boolean
): Page<AiAgentShowcaseResponse> {

    val user = userInfoProvider.currentUser()

    if (
        !user.roles.contains(CMS_ADMIN) &&
        showcaseRequestDto.userId != null &&
        showcaseRequestDto.userId != user.id
    ) {
        throw AiForbiddenException(
            errorCode = Metadata.ErrorMessages.FORBIDDEN_USER_ID_FILTER,
            message = messageProvider[Metadata.ErrorMessages.FORBIDDEN_USER_ID_FILTER]
        )
    }

    val calculateDeviations = shouldCalculateDeviations(
        userId = showcaseRequestDto.userId,
        roles = user.roles
    )

    /*
     * Фильтровать по отклонениям можно только тогда,
     * когда роль пользователя разрешает их рассчитывать.
     */
    val filterByDeviation = hasDeviation && calculateDeviations

    val sortField = when (showcaseRequestDto.sort) {
        AiAgentsSort.deadlineExpired -> "deadline_priority"
        AiAgentsSort.agentEffectOptimization -> "agent_effect_optimization"
        AiAgentsSort.alphabet -> "alphabet"
        null, AiAgentsSort.agentEffectRevenue -> "agent_effect_revenue"
    }

    val sortDirection = showcaseRequestDto.sortType.name

    val pageRequest = PageRequest.of(
        showcaseRequestDto.page!!,
        showcaseRequestDto.size!!,
        Sort.unsorted()
    )

    val initiativeIdsWithDeviation =
        if (filterByDeviation) {
            val filteredInitiativeIds =
                aiAgentRepository.findFilteredInitiativeIds(
                    search = showcaseRequestDto.search?.toSqlLikeExpression(),
                    terbankIds = showcaseRequestDto.terbankId ?: emptyList(),
                    programCodes = showcaseRequestDto.program,
                    agentStatusCodes = showcaseRequestDto.agentStatus,
                    blockCodes = showcaseRequestDto.block,
                    divisionCodes = showcaseRequestDto.division,
                    platformCodes = showcaseRequestDto.platform,
                    initiativeTypes = showcaseRequestDto.initiativeType,
                    userAudiences = showcaseRequestDto.userAudience,
                    disabled = showcaseRequestDto.disabled,
                    userId = showcaseRequestDto.userId,
                    deadlineExpiredIds = showcaseRequestDto.deadlineExpired?.map { it.name }
                )

            initiativeDeviationCalculator.findInitiativeIdsWithAnyDeviation(
                initiativeIds = filteredInitiativeIds
            )
        } else {
            emptySet()
        }

    if (filterByDeviation && initiativeIdsWithDeviation.isEmpty()) {
        return Page.empty(pageRequest)
    }

    val agentsPage = aiAgentRepository.findAll(
        search = showcaseRequestDto.search?.toSqlLikeExpression(),
        terbankIds = showcaseRequestDto.terbankId ?: emptyList(),
        programCodes = showcaseRequestDto.program,
        agentStatusCodes = showcaseRequestDto.agentStatus,
        blockCodes = showcaseRequestDto.block,
        divisionCodes = showcaseRequestDto.division,
        platformCodes = showcaseRequestDto.platform,
        initiativeTypes = showcaseRequestDto.initiativeType,
        userAudiences = showcaseRequestDto.userAudience,
        disabled = showcaseRequestDto.disabled,
        userId = showcaseRequestDto.userId,
        deadlineExpiredIds = showcaseRequestDto.deadlineExpired?.map { it.name },
        sortField = sortField,
        sortDirection = sortDirection,
        filterByDeviation = filterByDeviation,
        deviationInitiativeIds = initiativeIdsWithDeviation.ifEmpty { setOf(-1L) },
        pageRequest = pageRequest
    )

    val initiativeIds = agentsPage.content.map { it.id }.toSet()

    val metricAgentTypes = setOf(
        InitiativeMetricAgentType.AUTONOMOUS.value,
        InitiativeMetricAgentType.COPILOT.value
    )

    val initiativeIdsWithAgentTypes =
        if (initiativeIds.isEmpty()) {
            emptySet()
        } else {
            initiativeMetricTypeRepository.findInitiativeIdsWithAgentTypes(
                initiativeIds = initiativeIds,
                agentTypes = metricAgentTypes
            )
        }

    val deviationsByInitiativeId =
        if (calculateDeviations && initiativeIds.isNotEmpty()) {
            initiativeDeviationCalculator.calculate(
                initiativeIds = initiativeIds
            )
        } else {
            emptyMap()
        }

    return agentsPage.map { agent ->
        val hasMetricsValue =
            agent.agentStatus?.code in setOf(PILOT, TARGET_SOLUTION) &&
                agent.id in initiativeIdsWithAgentTypes

        agent.toAiAgentShowcaseResponse(
            hasMetricsValue = hasMetricsValue,
            deviations = deviationsByInitiativeId[agent.id].orEmpty()
        )
    }
}

private fun shouldCalculateDeviations(
    hasUserId: Boolean,
    roles: Set<String>
): Boolean {
    return if (hasUserId) {
        PROJECT_OFFICE in roles || CMS_ADMIN in roles
    } else {
        TRANSFORMATION_OFFICE in roles
    }
}

```
