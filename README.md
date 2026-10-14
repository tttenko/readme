```java


fun getAIAgentDetail(id: Long): AiAgentDetailResponse {
        val aiAgent = aiAgentRepository.findByIdOrNull(id)

        if (aiAgent == null || aiAgent.disabled == true) {
            throw AiResponseException(
                status = HttpStatus.NOT_FOUND,
                errorCode = AI_AGENT_NOT_FOUND,
                message = MessageFormat.format(messageProvider[AI_AGENT_NOT_FOUND])
            )
        }

        val userIds = aiAgent.agentContact.mapNotNull { it.userId }.toSet()

        val usersMap = if (userIds.isNotEmpty()) {
            try {
                authFeignClient.getUsers(ids = userIds, null).body?.associateBy { it.id }
            } catch (exception: Exception) {
                log.info(messageProvider[USER_NOT_FOUND])
                emptyMap()
            }
        } else {
            emptyMap()
        }

        var jiraUrl = aiAgent.agentJiraUrl?.let { jiraUrl ->
            when {
                jiraUrl.startsWith("CROSSGOALD") -> jiraService.getJiraDeltaUrl().plus(jiraUrl)
                jiraUrl.startsWith("CROSSGOAL") -> jiraService.getJiraSigmaUrl().plus(jiraUrl)
                else -> jiraUrl
            }
        }

        if (jiraUrl.isNullOrBlank()) {
            val jiraIssue = jiraIssueRepository.findByAgentIdAndTypeAndProject(aiAgent.id, "initiative", "crossgoal")
                .sortedBy { it.created }.firstOrNull()

            jiraUrl = jiraIssue?.jiraUrl

            if (jiraUrl.isNullOrBlank()) {
                jiraUrl = jiraIssue?.jiraKey
            }
        }

        val agentTypes = initiativeMetricTypeRepository.findAllByAiAgentId(aiAgent.id)
            .mapNotNull { it.agentType }
            .toSet()

        // Получение активных финансовых эффектов текущего года.
        val currentYear = LocalDate.now(ZoneId.of("Europe/Moscow")).year
        val financialEffects = financialEffectRepository.findActiveByInitiativeIdAndYear(aiAgent.id, currentYear)

        return aiAgent.toAiAgentDetailResponse(
            userAccountDtoMap = usersMap,
            jiraUrl = jiraUrl,
            agentTypes = agentTypes,
            financialEffects = financialEffects
        )
    }

    financialEffects: List<InitiativeFinancialEffectEntity> = emptyList()

    financialEffects = financialEffects.map { it.toShowcaseFinancialEffectResponse() }


```
