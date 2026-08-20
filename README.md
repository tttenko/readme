```java

@Component
class SyncFromJiraScheduler(
    private val strategyService: StrategyService,
    private val jiraService: JiraService,
    private val agentRepository: AIAgentRepository,
    private val qualityGateRepository: QualityGateRepository,
    private val statusRepository: StatusRepository,
    private val blockRepository: BlockRepository,
    private val enablerRepository: EnablerRepository,
    private val enablerNameNormalizer: EnablerNameNormalizer,
    private val optionsService: OptionsService,
    private val divisionRepository: DivisionRepository,
    private val initiativeTypeRepository: InitiativeTypeRepository,
    private val jiraIssueRepository: JiraIssueRepository,
    private val involvedResourceRepository: InvolvedResourceRepository,
    private val agentStatusSlaRepository: AgentStatusSlaRepository,
    private val aiagentQualityGateService: AiAgentQualityGateService,
    private val agentStrategyRepository: AgentStrategyRepository,
    private val jiraErrorTracker: JiraErrorTracker,
    private val syncFromJiraAgentTransactionRunner: SyncFromJiraAgentTransactionRunner,
) {
    companion object {
        private const val ZONE = "Europe/Moscow"
        const val SYNC_FROM_JIRA_SCHEDULER = "SyncFromJiraScheduler"

        private val JIRA_FIELDS = listOf(
            "summary",
            "issuetype",
            "description",
            "status",
            "customfield_16700",
            "customfield_16701",
            "labels",
            "customfield_30000",
            "customfield_30001",
            "customfield_30002",
            "customfield_34300",
            "customfield_30401",
            "customfield_31304",
            "customfield_31305",
            "customfield_31306",
            "customfield_31307",
            "issuelinks",
            "customfield_15903",
            "assignee",
            "reporter",
            "customfield_29202",
            "customfield_29203",
            "customfield_29205",
            "lastViewed",
            "resolutiondate",
            "created",
            "updated",
        )

        private val log by logger()
    }

    @Scheduled(cron = "\${scheduled.jira-sync.sync-update-cron}", zone = ZONE)
    @SchedulerLock(
        name = SYNC_FROM_JIRA_SCHEDULER,
        lockAtLeastFor = "\${scheduled.jira-sync.lock-at-least-for}",
        lockAtMostFor = "\${scheduled.jira-sync.lock-at-most-for}",
    )
    fun syncFromJira() {
        log.info("Starting sync from Jira")
        var page = 0
        var totalProcessed = 0
        var totalSkipped = 0
        var totalErrors = 0

        val options = optionsService.findAll().first()
        val maxResults = options.maxResults ?: 100

        while (true) {
            val agentsBatchPage = agentRepository.findAllWithPultIdAndCrossgoalPageable(
                PageRequest.of(page, maxResults)
            )

            for (candidate in agentsBatchPage.content) {
                try {
                    val issueKey = normalizeIssueKey(candidate.agentJiraUrl)
                    if (!issueKey.startsWith("CROSSGOAL-")) {
                        log.error("Invalid JIRA URL for agent ${candidate.id}: ${candidate.agentJiraUrl}")
                        totalSkipped++
                        continue
                    }
                    val jiraResponse = try {
                        jiraService.getIssue(
                            issueKey,
                            JIRA_FIELDS,
                        )
                    } catch (exception: Exception) {
                        val errorCount = jiraErrorTracker.increment()
                        log.error("Ошибка при получении issue $issueKey (Всего ошибок $errorCount): ${exception.message}")
                        null
                    }
                    log.info("На запрос с $issueKey получили ответ: id=${jiraResponse?.id}, key=${jiraResponse?.key}")

                    if (jiraResponse == null) {
                        log.error("Failed to get issue $issueKey after all retries")
                        totalErrors++

                        if (jiraErrorTracker.isErrorLimitExceeded()) {
                            log.error(
                                "Total Jira errors exceeded limit " +
                                        "(${jiraErrorTracker.getErrorCount()} > " +
                                        "${JiraErrorTracker.MAX_JIRA_ERRORS})"
                            )
                            return jiraErrorTracker.reset()
                        }
                        continue
                    }

                    val monitoringTasks = loadMonitoringTasks(
                        jiraResponse = jiraResponse,
                        maxResults = maxResults,
                    )

                    val processed = syncFromJiraAgentTransactionRunner.execute(candidate) { lockedAgent ->
                            if (
                                lockedAgent.disabled == true ||
                                normalizeIssueKey(lockedAgent.agentJiraUrl) != issueKey
                            ) {
                                log.info(
                                    "Skip agent ${lockedAgent.id}: " +
                                            "it was changed while Jira data was loading"
                                )
                                false
                            } else {
                                processAgent(
                                    agent = lockedAgent,
                                    jiraResponse = jiraResponse,
                                    monitoringTasks = monitoringTasks,
                                )
                                true
                            }
                        }
                    if (processed) {
                        totalProcessed++
                    } else {
                        totalSkipped++
                    }
                } catch (exception: Exception) {
                    log.error(
                        "Unexpected error processing agent ${candidate.id}: " +
                                exception.message,
                        exception,
                    )
                    totalErrors++
                }
            }
            if (agentsBatchPage.isLast) {
                break
            }
            page++
        }

        jiraErrorTracker.reset()
        log.info(
            "Sync from Jira completed. " +
                    "Processed: $totalProcessed, " +
                    "Skipped: $totalSkipped, " +
                    "Errors: $totalErrors"
        )
    }

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
                    customfield_30000 =
                    jiraResponse.fields?.customfield_30000,
                    customfield_30001 =
                    jiraResponse.fields?.customfield_30001,
                    customfield_30002 =
                    jiraResponse.fields?.customfield_30002,
                    enabledDivisions = enabledDivisions,
                    enabledBlocks = enabledBlocks,
                )

            division = resolvedDivision
            block = resolvedBlock

            initiativeType = determineInitiativeType(
                labels = jiraResponse.fields?.labels,
                initiativeTypes = initiativeTypes,
            )

            agentEffectOptimization = findFirstBigDecimalInText(
                jiraResponse.fields?.customfield_34300
            )
            agentEffectRevenue = findFirstBigDecimalInText(
                jiraResponse.fields?.customfield_30401
            )

            jiraFromStatus = "inProgress"
            updated = LocalDateTime.now()
        }

        agentRepository.save(agent)

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
        log.debug("В Пульт синхронизирована с Jira инициатива с ключом ${jiraResponse.key}")
    }


    private fun loadMonitoringTasks(
        jiraResponse: IssueDto,
        maxResults: Int,
    ): List<SearchIssueDto>? {
        if (hasMonitoringLabel(jiraResponse)) {
            return null
        }

        val monitoringLink = findMonitoringLink(jiraResponse) ?: return null

        val jiraKey = monitoringLink.inwardIssue?.key
                ?: monitoringLink.outwardIssue?.key

        if (jiraKey.isNullOrBlank()) {
            return null
        }

        return try {
            jiraService.searchTasksByEpicKey(
                jiraKey,
                maxResults,
            )
        } catch (exception: Exception) {
            val errorCount = jiraErrorTracker.increment()

            log.error("Ошибка получения списка задач для epic $jiraKey " +
                        "(Всего ошибок $errorCount): ${exception.message}")
            emptyList()
        }
    }

    private fun hasMonitoringLabel(
        jiraResponse: IssueDto,
    ): Boolean {
        return jiraResponse.fields
            ?.labels
            ?.let { labels ->
                "AI-эффективность" in labels
            } == true
    }

    private fun findMonitoringLink(
        jiraResponse: IssueDto,
    ): GetIssueLinkResponse? {
        return jiraResponse.fields
            ?.issuelinks
            ?.find { link ->
                listOfNotNull(
                    link.outwardIssue?.fields?.summary,
                    link.inwardIssue?.fields?.summary,
                )
                    .map { summary -> summary.lowercase() }
                    .any { summary -> "мониторинг" in summary }
            }
    }

    private fun createInitiativeJiraIssue(
        agent: AIAgentEntity,
        jiraResponse: IssueDto,
    ) {
        val jiraKey = jiraResponse.key
            ?.takeIf { it.isNotBlank() }
            ?: run {
                log.warn("Cannot create initiative jira_issue for agent ${agent.id}: Jira key is empty")
                return
            }

        val existingIssues = jiraIssueRepository.findByAgentIdAndTypeAndProject(
            agentId = agent.id,
            type = JiraIssueType.initiative.name,
            project = "crossgoal",
        )

        val jiraIssue = existingIssues.firstOrNull()
            ?: JiraIssueEntity(
                agent = agent,
                type = JiraIssueType.initiative.name,
                project = "crossgoal",
            )

        jiraIssue.apply {
            jiraId = jiraResponse.id
            this.jiraKey = jiraKey
            jiraUrl = jiraService.getJiraSigmaUrl() + jiraKey
        }

        jiraIssueRepository.save(jiraIssue)
    }

    private fun normalizeIssueKey(jiraUrl: String?): String {
        if (jiraUrl == null) {
            return ""
        }

        if (jiraUrl.startsWith("CROSSGOAL-")) {
            return jiraUrl
        }

        val index = jiraUrl.lastIndexOf("CROSSGOAL-")
        return if (index >= 0) {
            jiraUrl.substring(index)
        } else {
            jiraUrl
        }
    }

    private fun extractJiraIssueKey(value: String?): String? {
        return value
            ?.uppercase()
            ?.let { Regex("""[A-Z][A-Z0-9]+-\d+""").find(it)?.value }
    }

    private fun determineDivisionAndBlock(
        customfield_30000: List<String>?,
        customfield_30001: List<String>?,
        customfield_30002: List<String>?,
        enabledDivisions: List<DivisionEntity>,
        enabledBlocks: List<BlockEntity>,
    ): Pair<DivisionEntity?, BlockEntity?> {
        val divisionBy30001 = customfield_30001?.let { label ->
            enabledDivisions.find { label.contains(it.label) }
        }
        if (divisionBy30001 != null) {
            return Pair(divisionBy30001, divisionBy30001.block)
        }

        val divisionBy30002 = customfield_30002?.let { label ->
            enabledDivisions.find { label.contains(it.label) }
        }
        if (divisionBy30002 != null) {
            return Pair(divisionBy30002, divisionBy30002.block)
        }

        val divisionBy30000 = customfield_30000?.let { label ->
            enabledDivisions.find { label.contains(it.label) }
        }
        if (divisionBy30000 != null) {
            return Pair(divisionBy30000, divisionBy30000.block)
        }

        val blockBy30001 = customfield_30001?.let { label ->
            enabledBlocks.find { label.contains(it.label) }
        }
        if (blockBy30001 != null) {
            return Pair(null, blockBy30001)
        }

        val blockBy30002 = customfield_30002?.let { label ->
            enabledBlocks.find { label.contains(it.label) }
        }
        if (blockBy30002 != null) {
            return Pair(null, blockBy30002)
        }

        val blockBy30000 = customfield_30000?.let { label ->
            enabledBlocks.find { label.contains(it.label) }
        }
        if (blockBy30000 != null) {
            return Pair(null, blockBy30000)
        }

        val cibBlock = enabledBlocks.find { it.code == "cib" }
        return Pair(null, cibBlock)
    }

    private fun determineInitiativeType(
        labels: List<String>?,
        initiativeTypes: List<InitiativeTypeEntity>,
    ): InitiativeTypeEntity? {
        val hasAgentLabel = labels?.map { it.lowercase() }?.any { "агент" in it } ?: false
        return initiativeTypes.find { it.code == if (hasAgentLabel) "agent" else "genAiSolution" }
    }

    private fun updateAgentStrategies(
        agent: AIAgentEntity,
        strategies: List<StrategyEntity>,
        jiraResponse: IssueDto,
    ) {
        val issueLinks = jiraResponse.fields?.issuelinks
            ?: run {
                log.debug("Issuelinks not found in Jira response for agent ${agent.id}, skip strategy sync")
                return
            }

        val jiraKeysFromResponse = issueLinks
            .flatMap {
                listOfNotNull(
                    extractJiraIssueKey(it.outwardIssue?.key),
                    extractJiraIssueKey(it.inwardIssue?.key),
                )
            }
            .toSet()

        val strategiesToUpdate = strategies
            .filter { strategy ->
                extractJiraIssueKey(strategy.jiraIssue) in jiraKeysFromResponse
            }.associateBy { it.id }

        log.debug("Для агента ${agent.id} найдены стратегии для обновления ${strategiesToUpdate.keys.joinToString()} по Jira issue key ${jiraKeysFromResponse.joinToString()}")

        val currentAgentStrategies = agentStrategyRepository.findAllByAgentId(agent.id)
        val agentStrategiesToDelete = currentAgentStrategies
            .filter { agentStrategy -> agentStrategy.strategy?.id !in strategiesToUpdate.keys }

        if (agentStrategiesToDelete.isNotEmpty()) {
            agentStrategyRepository.deleteAll(agentStrategiesToDelete)
        }

        val agentStrategiesToUpdate = strategiesToUpdate.values.map { strategy ->
            val agentStrategy =
                currentAgentStrategies.find { it.strategy?.id == strategy.id }
                ?: AgentStrategyEntity (
                    agent = agent,
                    strategy = strategy
                )
            agentStrategy.apply { jiraLink = "done" }
        }
        agentStrategyRepository.saveAll(agentStrategiesToUpdate)
    }

    private fun syncWithMonitoringEpic(
        agent: AIAgentEntity,
        jiraResponse: IssueDto,
        qualityGates: List<QualityGateEntity>,
        monitoringTasks: List<SearchIssueDto>?,
    ) {
        val hasMonitoringLabel = hasMonitoringLabel(jiraResponse)
        log.info("В запросе есть label \"AI-эффективность\" - $hasMonitoringLabel")
        val monitoringLink = findMonitoringLink(jiraResponse)
        log.info("В запросе есть label \"мониторинг\" - ${monitoringLink != null}")

        if (monitoringLink != null && !hasMonitoringLabel) {
            createOrUpdateMonitoringConnection(
                agent = agent,
                monitoringLink = monitoringLink,
                jiraResponse = jiraResponse,
                qualityGates = qualityGates,
                relatedTasks = monitoringTasks.orEmpty(),
            )
        } else {
            agent.jiraFromStatus = "done"
            agent.jiraFromUpdated = LocalDateTime.now()
            agent.updated = LocalDateTime.now()
            agentRepository.save(agent)
            log.debug("В Пульт синхронизирована с Jira инициатива с ключом ${jiraResponse.key}")
        }
    }

    private fun createOrUpdateMonitoringConnection(
        agent: AIAgentEntity,
        monitoringLink: GetIssueLinkResponse,
        jiraResponse: IssueDto,
        qualityGates: List<QualityGateEntity>,
        relatedTasks: List<SearchIssueDto>,
    ) {
        val jiraId =
            monitoringLink.inwardIssue?.id
                ?: monitoringLink.outwardIssue?.id

        val jiraKey =
            monitoringLink.inwardIssue?.key
                ?: monitoringLink.outwardIssue?.key

        val epic = jiraIssueRepository
            .findByAgentIdAndTypeAndProject(agent.id, JiraIssueType.epic.name, "crossgoal")
            .firstOrNull()
            ?: JiraIssueEntity(
                agent = agent,
                type = JiraIssueType.epic.name,
                project = "crossgoal",
            )

        epic.apply {
            this.jiraId = jiraId
            this.jiraKey = jiraKey
            jiraUrl = jiraService.getJiraSigmaUrl() + jiraKey
        }

        jiraIssueRepository.save(epic)
        if (epic.jiraKey.isNullOrBlank()) {
            return
        }

        log.info("Найдено ${relatedTasks.size} jira задач по запросу search c jiraKey ${epic.jiraKey}")

        updateQualityGatesFromTasks(
            agent = agent,
            tasks = relatedTasks,
            qualityGates = qualityGates.filter {
                it.type == QualityGateType.quality_gate
            },
        )

        updateAgentStatusSlaFromTasks(
            agent = agent,
            tasks = relatedTasks,
            qualityGates = qualityGates,
        )

        updateJiraIssuesForTasks(
            agent = agent,
            tasks = relatedTasks,
            qualityGates = qualityGates,
        )

        agent.jiraFromStatus = "done"
        agent.jiraUpdated = LocalDateTime.now()
        agent.updated = LocalDateTime.now()
        agentRepository.save(agent)

        log.debug("В Пульт синхронизирована с Jira инициатива с ключом ${jiraResponse.key}")
    }

    private fun updateInvolvedResources(agent: AIAgentEntity, jiraResponse: IssueDto) {
        val resources = mutableListOf<InvolvedResourceEntity>()

        val slaMappings = listOf(
            Triple(jiraResponse.fields?.customfield_31304, "without_steerCo", "business"),
            Triple(jiraResponse.fields?.customfield_31305, "without_steerCo", "it"),
            Triple(jiraResponse.fields?.customfield_31306, "steerCo", "business"),
            Triple(jiraResponse.fields?.customfield_31307, "steerCo", "it"),
        )

        for ((fieldValue, source, type) in slaMappings) {
            val numericValue = findFirstBigDecimalInText(fieldValue)
            if (numericValue != null) {
                resources.add(
                    InvolvedResourceEntity().apply {
                        id = InvolvedResourceEmbeddedId(
                            aiAgentId = agent.id,
                            source = source,
                            type = type,
                        )
                        value = numericValue
                        timeAllocated = null
                        updated = LocalDateTime.now()
                        this.aiAgent = agent
                    }
                )
            } else if (!fieldValue.isNullOrBlank()) {
                log.warn("Failed to parse numeric value from SLA field for agent ${agent.id}: field='$fieldValue'")
            }
        }


        involvedResourceRepository.deleteAllByAiAgentId(agent.id)
        if (resources.isNotEmpty()) {
            involvedResourceRepository.saveAll(resources)
            log.debug("Saved ${resources.size} involved resources for agent ${agent.id}")
        }
    }

    private fun updateEnablers(
        agent: AIAgentEntity,
        jiraResponse: IssueDto,
        enablersByNormalizedName: Map<String, EnablerEntity>,
    ) {
        val selectedEnablers = jiraResponse.fields
            ?.customfield_15903
            ?.filter { it.checked == true }
            ?: emptyList()

        val agentEnablers = mutableSetOf<EnablerEntity>()

        for (option in selectedEnablers) {
            val normalizedName = enablerNameNormalizer.normalize(option.name)

            if (normalizedName == null) {
                log.debug("Пустое название энейблера в customfield_15903 для агента ${agent.id}")
                continue
            }

            val enabler = enablersByNormalizedName[normalizedName]

            if (enabler == null) {
                log.debug(
                    "Энейблер '${option.name}' не найден в справочнике enabler после нормализации '$normalizedName'"
                )
                continue
            }

            agentEnablers.add(enabler)
        }

        enablerRepository.deleteAllByAgentId(agent.id)
        if (agentEnablers.isNotEmpty()) {
            val enablerIds = agentEnablers.map { it.id }
            enablerRepository.addAllToAgent(agent.id, enablerIds)
        }
    }

    private fun updateGigausage(agent: AIAgentEntity, jiraResponse: IssueDto) {
        val gigausageLink = jiraResponse.fields?.issuelinks
            ?.flatMap { setOf(it.outwardIssue, it.inwardIssue) }
            ?.filterNotNull()
            ?.filter { it.key?.lowercase()?.let { "gigausage" in it } == true }
            ?.maxByOrNull { it.id ?: "-1" }

        if (gigausageLink?.key != null) {
            val gigausageIssues = jiraIssueRepository.findByAgentIdAndTypeAndProject(
                agent.id,
                "initiative",
                "gigausage",
            )

            val gigausageIssue = gigausageIssues.firstOrNull()
                ?: JiraIssueEntity(agent = agent, type = "initiative", project = "gigausage")

            gigausageIssue.jiraKey = gigausageLink.key
            gigausageIssue.jiraId = gigausageLink.id
            gigausageIssue.jiraUrl = "${jiraService.getJiraSigmaUrl()}${gigausageLink.key}"

            jiraIssueRepository.save(gigausageIssue)
            log.info("Создана/Обновлена связь с GIGAUSAGE: ${gigausageLink.key}")
        }
    }

    private fun updateQualityGatesFromTasks(
        agent: AIAgentEntity,
        tasks: List<SearchIssueDto>,
        qualityGates: List<QualityGateEntity>,
    ) {
        val taskQualityGateList = mutableListOf<SearchIssueDto>()

        tasks.forEach { task ->
            val summary = task.fields?.summary?.lowercase() ?: return@forEach
            val matchingQualityGate = qualityGates.find { qg ->
                qg.regexp?.let { Regex(it).matches(summary) } ?: false
            }
            if (matchingQualityGate != null) {
                taskQualityGateList.add(task)
                val agentQualityGate = agent.qualityGates.find { it.qualityGate?.code == matchingQualityGate.code }
                    ?: AIAgentQualityGateEntity().apply {
                        primaryKey = AIAgentQualityGatePK().apply {
                            aiAgentId = agent.id
                            qualityGateCode = matchingQualityGate.code
                        }
                        this.qualityGate = matchingQualityGate
                        this.agent = agent
                    }

                val state: QualityGateState = when (task.fields.status?.id) {
                    "10110", "5", "14103" -> QualityGateState.checked
                    else -> QualityGateState.unchecked
                }
                aiagentQualityGateService.updateState(agentQualityGate, state)
            } else {
                log.debug("Qua ${task.key}")
            }
        }

        val qualityGateTasks = taskQualityGateList.filter { task ->
            task.fields?.status?.id != null
        }

        if (qualityGateTasks.isNotEmpty()) {
            val epicStatusId = qualityGateTasks.first().fields?.status?.id
            if (epicStatusId != null) {
                log.debug("Статус эпика мониторинга определен: $epicStatusId")
            }
        }
    }

    private fun updateAgentStatusSlaFromTasks(
        agent: AIAgentEntity,
        tasks: List<SearchIssueDto>,
        qualityGates: List<QualityGateEntity>,
    ) {
        val stageTasks = tasks.filter { task ->
            task.fields?.status != null
        }

        val inProgressStageStatuses = mutableListOf<Pair<Int, String>>()
        val todoStageStatuses = mutableListOf<Pair<Int, String>>()
        val statusQualityGates = qualityGates.filter { it.type == QualityGateType.status }

        stageTasks.forEach { task ->
            val statusId = task.fields?.status?.id ?: return@forEach

            val matchingQualityGate = findQualityGateByTaskSummary(
                summary = task.fields.summary,
                qualityGates = statusQualityGates,
            )

            if (matchingQualityGate != null) {
                val statusCode = matchingQualityGate.status?.code ?: return@forEach
                val ordering = matchingQualityGate.ordering ?: 0

                if (matchingQualityGate.type == QualityGateType.status) {
                    when (statusId) {
                        "3" -> inProgressStageStatuses.add(ordering to statusCode)
                        "10109", "4", "10501" -> todoStageStatuses.add(ordering to statusCode)
                    }

                    val plannedDate = jiraResponseToSlaDate(task.fields.customfield_16701)
                    val completedDate = jiraResponseToSlaDate(task.fields.resolutiondate)

                    val status = statusRepository.findFirstByCode(statusCode)
                    if (status != null) {
                        val agentSla = agent.agentStatusSla.find {
                            it.agentStatus?.code == statusCode
                        } ?: AgentStatusSlaEntity().apply {
                            agentStatus = status
                            aiAgent = agent
                        }

                        agentSla.plannedDate = plannedDate
                        agentSla.completedDate = completedDate
                        agentStatusSlaRepository.save(agentSla)
                    }
                }
            }
        }

        val currentStatusCode = when {
            inProgressStageStatuses.isNotEmpty() -> {
                inProgressStageStatuses
                    .maxByOrNull { it.first }
                    ?.second
            }

            todoStageStatuses.isNotEmpty() -> {
                todoStageStatuses
                    .minByOrNull { it.first }
                    ?.second
            } else -> {
                "targetSolution"
            }
        }

        if (currentStatusCode != null) {
            log.debug("Текущий статус агента определен: $currentStatusCode")

            val currentStatus = statusRepository.findFirstByCode(currentStatusCode)
            if (currentStatus != null) {
                agent.agentStatus = currentStatus
            }

            agent.jiraFromStatus = currentStatusCode
            agent.jiraUpdated = LocalDateTime.now()
            agent.updated = LocalDateTime.now()
            agentRepository.save(agent)
        }
    }

    private fun jiraResponseToSlaDate(dateString: String?): LocalDateTime? {
        if (dateString.isNullOrBlank()) {
            return null
        }

        return try {
            OffsetDateTime.parse(
                dateString,
                DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mm:ss.SSSZ")
            ).toLocalDateTime()
        } catch (e: Exception) {
            try {
                OffsetDateTime.parse(
                    dateString,
                    DateTimeFormatter.ISO_OFFSET_DATE_TIME
                ).toLocalDateTime()
            } catch (e2: Exception) {
                try {
                    LocalDate.parse(
                        dateString,
                        DateTimeFormatter.ISO_LOCAL_DATE
                    ).atStartOfDay()
                } catch (e3: Exception) {
                    null
                }
            }
        }
    }

    private fun updateJiraIssuesForTasks(
        agent: AIAgentEntity,
        tasks: List<SearchIssueDto>,
        qualityGates: List<QualityGateEntity>,
    ) {
        val epic = jiraIssueRepository
            .findByAgentIdAndTypeAndProject(agent.id, "epic", "crossgoal")
            .firstOrNull()

        val jiraKeys = tasks
            .mapNotNull { it.key }
            .toSet()

        val existingIssuesByJiraKey: MutableMap<String, JiraIssueEntity> =
            if (jiraKeys.isEmpty()) {
                mutableMapOf()
            } else {
                jiraIssueRepository.findAllByAgentIdAndJiraKeyIn(agent.id, jiraKeys)
                    .mapNotNull { jiraIssue ->
                        jiraIssue.jiraKey?.let{ it to jiraIssue}
                    }
                    .toMap()
                    .toMutableMap()
            }

        val issuesToSave = mutableListOf<JiraIssueEntity>()

        tasks.forEach { task ->
            val jiraKey = task.key ?: return@forEach

            val matchingQualityGate = findQualityGateByTaskSummary(
                summary = task.fields?.summary,
                qualityGates = qualityGates,
            )

            if (matchingQualityGate == null) {
                log.debug("Quality gate not found for Jira task $jiraKey, summary='${task.fields?.summary}'")
                return@forEach
            }

            val jiraIssue = existingIssuesByJiraKey[jiraKey]
                ?: JiraIssueEntity().apply {
                    this.agent = agent
                    this.project = "crossgoal"
                    this.type = "task"
                }.also { newIssue ->
                    existingIssuesByJiraKey[jiraKey] = newIssue
                }

            jiraIssue.apply {
                this.parentId = epic?.id
                this.jiraId = task.id
                this.jiraKey = jiraKey
                this.jiraUrl = "${jiraService.getJiraSigmaUrl()}$jiraKey"
                this.qualityGate = matchingQualityGate
            }
            issuesToSave.add(jiraIssue)
        }
        if (issuesToSave.isNotEmpty()) {
            jiraIssueRepository.saveAll(issuesToSave)
        }
    }

    private fun findQualityGateByTaskSummary(
        summary: String?,
        qualityGates: List<QualityGateEntity>,
    ): QualityGateEntity? {
        val normalizedSummary = normalizeJiraText(summary) ?: return null

        return qualityGates.firstOrNull { qualityGate ->
            val labels = qualityGate.regexp
                ?.split(",")
                ?.mapNotNull { normalizeJiraText(it) }
                .orEmpty()

            labels.isNotEmpty() && labels.all { label ->
                normalizedSummary.contains(label)
            }
        } ?: qualityGates.firstOrNull { qualityGate ->
            normalizeJiraText(qualityGate.name)
                ?.let { qualityGateName -> normalizedSummary.contains(qualityGateName) }
                ?: false
        }
    }

    private fun normalizeJiraText(value: String?): String? {
        return value
            ?.lowercase()
            ?.replace(Regex("\\s+"), " ")
            ?.trim()
            ?.takeIf { it.isNotBlank() }
    }

    private fun findFirstBigDecimalInText(value: String?): BigDecimal? {
        if (value.isNullOrBlank()) {
            return null
        }

        return Regex("""-?\d+(?:[.,]\d+)?""")
            .find(value)
            ?.value
            ?.replace(",", ".")
            ?.toBigDecimalOrNull()
    }

}

```
