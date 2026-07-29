```java

fun getAIAgentList(
    showcaseRequestDto: ShowcaseRequestDto,
    hasDeviation: Boolean,
): Page<AiAgentShowcaseResponse> {

    val user = userInfoProvider.currentUser()
    if (!user.roles.contains("CMS_ADMIN") &&
        (showcaseRequestDto.userId != null) &&
        (showcaseRequestDto.userId != user.id)
    ) {
        throw AiForbiddenException(
            errorCode = Metadata.ErrorMessages.FORBIDDEN_USER_ID_FILTER,
            message = messageProvider[
                Metadata.ErrorMessages.FORBIDDEN_USER_ID_FILTER
            ]
        )
    }

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
        if (hasDeviation) {
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
                    deadlineExpiredIds =
                        showcaseRequestDto.deadlineExpired?.map { it.name }
                )

            initiativeDeviationCalculator.findInitiativeIdsWithAnyDeviation(
                initiativeIds = filteredInitiativeIds
            )
        } else {
            emptySet()
        }

    if (hasDeviation && initiativeIdsWithDeviation.isEmpty()) {
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
        filterByDeviation = hasDeviation,
        deviationInitiativeIds =
            if (initiativeIdsWithDeviation.isEmpty()) {
                setOf(-1L)
            } else {
                initiativeIdsWithDeviation
            },
        pageRequest = pageRequest
    )

    val initiativeIds = agentsPage.content
        .map { it.id }
        .toSet()

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
        initiativeDeviationCalculator.calculate(
            initiativeIds = initiativeIds
        )

    return agentsPage.map { agent ->
        val hasMetricsValue =
            agent.agentStatus?.code == TARGET_SOLUTION &&
                agent.id in initiativeIdsWithAgentTypes

        agent.toAiAgentShowcaseResponse(
            hasMetricsValue = hasMetricsValue,
            deviations = deviationsByInitiativeId[agent.id].orEmpty()
        )
    }
}



```
