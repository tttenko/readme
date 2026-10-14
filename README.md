```java


data class ShowcaseFinancialEffectResponse(

    @field:Schema(
        description = "Идентификатор карточки финансового эффекта",
        type = "integer",
        format = "int64",
        example = "125"
    )
    val id: Long,

    @field:Schema(
        description = "Технический код типа финансового эффекта",
        type = "string",
        maxLength = 60,
        example = "GROUP_REVENUE_GROWTH"
    )
    val typeCode: String,

    @field:Schema(
        description = "Наименование типа финансового эффекта",
        type = "string",
        maxLength = 255,
        example = "Рост доходов Группы"
    )
    val typeName: String,

    @field:Schema(
        description = "Единица измерения финансового эффекта",
        type = "string",
        example = "RUB"
    )
    val unit: String,

    @field:Schema(
        description = "Направление достижения плана: true — больше лучше, false — меньше лучше",
        type = "boolean",
        example = "true"
    )
    val moreIsBetter: Boolean,

    @field:Schema(
        description = "Плановое значение финансового эффекта",
        type = "string",
        example = "15000000"
    )
    val plan: String,

    @field:Schema(
        description = "Фактическое значение финансового эффекта",
        type = "string",
        nullable = true,
        example = "12500000"
    )
    val fact: String?,

    @field:Schema(
        description = "Месяц и год достижения финансового эффекта",
        type = "string",
        pattern = "^\\d{4}-(0[1-9]|1[0-2])$",
        example = "2026-12"
    )
    val targetMonth: String,

    @field:ArraySchema(
        arraySchema = Schema(description = "Ссылки на согласование финансового эффекта"),
        schema = Schema(type = "string", example = "https://example.com/approval")
    )
    val approvalLinks: List<String>,

    @field:Schema(
        description = "Дата и время создания карточки финансового эффекта",
        type = "string",
        format = "date-time",
        example = "2026-05-01T10:00:00Z"
    )
    val createdAt: Instant
)

field:ArraySchema(schema = Schema(implementation = ShowcaseFinancialEffectResponse::class))
val financialEffects: List<ShowcaseFinancialEffectResponse> = emptyList()

/**
 * Возвращает активные финансовые эффекты текущего года для инициатив страницы Showcase.
 * Карточки сортируются по дате создания и идентификатору.
 */
@Query(
    """
    SELECT financialEffect
    FROM InitiativeFinancialEffectEntity financialEffect
    JOIN FETCH financialEffect.type
    WHERE financialEffect.initiative.id IN :initiativeIds
      AND financialEffect.effectYear = :year
      AND financialEffect.disabled = false
    ORDER BY financialEffect.initiative.id ASC, financialEffect.created ASC, financialEffect.id ASC
    """
)
fun findActiveByInitiativeIdsAndYear(
    @Param("initiativeIds") initiativeIds: Collection<Long>,
    @Param("year") year: Int
): List<InitiativeFinancialEffectEntity>

fun InitiativeFinancialEffectEntity.toShowcaseFinancialEffectResponse(): ShowcaseFinancialEffectResponse {
    val financialEffectType = type!!
    val plan = planValue!!
    val targetMonth = targetMonth!!
    val createdAt = created!!

    return ShowcaseFinancialEffectResponse(
        id = id,
        typeCode = financialEffectType.code,
        typeName = financialEffectType.name,
        unit = financialEffectType.unit,
        moreIsBetter = financialEffectType.moreIsBetter,
        plan = plan.toPlainString(),
        fact = factValue?.toPlainString(),
        targetMonth = YearMonth.from(targetMonth).toString(),
        approvalLinks = approvalLinks?.map { it.asText() }.orEmpty(),
        createdAt = createdAt.atZone(ZoneId.systemDefault()).toInstant()
    )
}

fun getAIAgentList(showcaseRequestDto: ShowcaseRequestDto): Page<AiAgentShowcaseResponse> {
    val user = userInfoProvider.currentUser()

    if (!user.roles.contains("CMS_ADMIN") && showcaseRequestDto.userId != null && showcaseRequestDto.userId != user.id) {
        throw AiForbiddenException(
            errorCode = Metadata.ErrorMessages.FORBIDDEN_USER_ID_FILTER,
            message = messageProvider[Metadata.ErrorMessages.FORBIDDEN_USER_ID_FILTER]
        )
    }

    val filterByDeviation = showcaseRequestDto.hasDeviation

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
            val filteredInitiativeIds = aiAgentRepository.findFilteredInitiativeIds(
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

    val crossgoalJiraUrlsByInitiativeId = if (initiativeIds.isEmpty()) {
        emptyMap()
    } else {
        jiraIssueRepository.findCrossgoalJiraUrls(initiativeIds = initiativeIds)
            .associate { jiraIssue -> jiraIssue.agentId to jiraIssue.jiraUrl }
    }

    val metricAgentTypes = setOf(
        InitiativeMetricAgentType.AUTONOMOUS.value,
        InitiativeMetricAgentType.COPILOT.value
    )

    val initiativeIdsWithAgentTypes = if (initiativeIds.isEmpty()) {
        emptySet()
    } else {
        initiativeMetricTypeRepository.findInitiativeIdsWithAgentTypes(
            initiativeIds = initiativeIds,
            agentTypes = metricAgentTypes
        )
    }

    val deviationsByInitiativeId = if (initiativeIds.isNotEmpty()) {
        initiativeDeviationCalculator.calculate(initiativeIds = initiativeIds)
    } else {
        emptyMap()
    }

    // Получение финансовых эффектов текущего года для всех инициатив страницы.
    val financialEffectsByInitiativeId = if (initiativeIds.isEmpty()) {
        emptyMap()
    } else {
        val currentYear = LocalDate.now(ZoneId.of("Europe/Moscow")).year

        financialEffectRepository.findActiveByInitiativeIdsAndYear(initiativeIds, currentYear)
            .groupBy { financialEffect -> financialEffect.initiative!!.id }
    }

    return agentsPage.map { agent ->
        val hasMetricsValue =
            agent.agentStatus?.code in setOf(PILOT, TARGET_SOLUTION) &&
                agent.id in initiativeIdsWithAgentTypes

        agent.toAiAgentShowcaseResponse(
            hasMetricsValue = hasMetricsValue,
            deviations = deviationsByInitiativeId[agent.id].orEmpty(),
            crossgoalJiraUrl = crossgoalJiraUrlsByInitiativeId[agent.id],
            financialEffects = financialEffectsByInitiativeId[agent.id].orEmpty()
        )
    }
}


```
