```java

/**
 * Формирует запросы поиска изменённых Task и инициатив для FR2.
 */
@Component
class JiraUpdateSearchRequestFactory {

    companion object {
        private val TASK_FIELDS = listOf(
            "summary", "description", "status", "customfield_16700", "customfield_16701",
            "assignee", "reporter", "lastViewed", "resolutiondate", "created", "updated"
        )

        private val INITIATIVE_FIELDS = listOf(
            "summary", "description", "status", "labels",
            "customfield_30000", "customfield_30001", "customfield_30002",
            "customfield_34300", "customfield_30401",
            "customfield_31304", "customfield_31305", "customfield_31306", "customfield_31307",
            "issuelinks", "customfield_15903", "assignee", "reporter",
            "customfield_29202", "customfield_29203", "customfield_29205",
            "lastViewed", "resolutiondate", "created", "updated"
        )
    }

    /** Ищет изменённые Task эпика мониторинга. */
    fun createUpdatedTasksRequest(updateDepth: Int, maxResults: Int, startAt: Int): SearchIssueRequestDto {
        require(updateDepth > 0) { "Jira updateDepth must be positive" }
        require(maxResults > 0) { "Jira maxResults must be positive" }

        return SearchIssueRequestDto(
            fields = TASK_FIELDS,
            jql = "project = CROSSGOAL AND issuetype = Task " +
                "AND \"Epic Link\"=\"Мониторинг портфеля AI-Native\" " +
                "AND updated >= -${updateDepth}d ORDER BY updated DESC",
            maxResults = maxResults,
            startAt = startAt
        )
    }

    /**
     * Ищет обновлённые инициативы, включая отменённые и ClassicML.
     *
     * У отменённой инициативы resolution может перестать быть Unresolved.
     * Метка ClassicML может остаться без исходной метки портфеля.
     */
    fun createUpdatedInitiativesRequest(updateDepth: Int, maxResults: Int, startAt: Int): SearchIssueRequestDto {
        require(updateDepth > 0) { "Jira updateDepth must be positive" }
        require(maxResults > 0) { "Jira maxResults must be positive" }

        return SearchIssueRequestDto(
            fields = INITIATIVE_FIELDS,
            jql = "project = CROSSGOAL AND issuetype = Инициатива " +
                "AND ((resolution = Unresolved AND labels IN (AI_Native_портфель, \"AI-эффективность\")) " +
                "OR labels = ClassicML OR status = \"Отменена\") " +
                "AND updated >= -${updateDepth}d ORDER BY updated DESC",
            maxResults = maxResults,
            startAt = startAt
        )
    }
}

/** Найденная связь инициативы Пульта с ключом CROSSGOAL. */
interface JiraUpdatedInitiativeReference {
    val agentId: Long
    val jiraKey: String
}

/**
 * Находит существующие инициативы по ключам Jira одной страницы Search.
 */
@Repository
interface JiraUpdatedInitiativeRepository : JpaRepository<AIAgentEntity, Long> {

    @Query(
        value = """
            select distinct issue.agent_id as "agentId", upper(issue.jira_key) as "jiraKey"
            from jira_issue issue
            where issue.type = 'initiative'
              and lower(issue.project) = 'crossgoal'
              and upper(issue.jira_key) in (:jiraKeys)

            union

            select agent.id as "agentId",
                   substring(upper(agent.agent_jira_url) from '(CROSSGOAL-[0-9]+)') as "jiraKey"
            from ai_agent agent
            where substring(upper(agent.agent_jira_url) from '(CROSSGOAL-[0-9]+)') in (:jiraKeys)
        """,
        nativeQuery = true
    )
    fun findInitiativeReferences(@Param("jiraKeys") jiraKeys: Collection<String>): List<JiraUpdatedInitiativeReference>
}

enum class JiraUpdatedInitiativeResult {
    UPDATED,
    SKIPPED
}

/**
 * Обновляет разрешённые основные поля инициативы.
 *
 * Описание, контакты, текущий бизнес-статус и связанные сущности
 * на этом этапе не изменяются.
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
     * Для инициатив, обработанных Task-подпроцессом этого запуска,
     * используется значение updated до обработки Task.
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

        if (pultUpdated != null && pultUpdated.isAfter(jiraUpdated)) {
            log.debug("Skipping initiative updated later in Pult: agentId={}, jiraKey={}", agentId, issue.key)
            return JiraUpdatedInitiativeResult.SKIPPED
        }

        val fields = requireNotNull(issue.fields) { "Jira initiative fields are missing: ${issue.key}" }
        val summary = fields.summary?.takeIf(String::isNotBlank)
            ?: error("Jira initiative summary is missing: ${issue.key}")

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

/**
 * Обрабатывает найденные обновления существующих Jira-инициатив.
 */
@Service
class JiraUpdatedInitiativeService(
    private val searchRequestFactory: JiraUpdateSearchRequestFactory,
    private val jiraSearchPaginator: JiraSearchPaginator,
    private val initiativeRepository: JiraUpdatedInitiativeRepository,
    private val initiativePersistenceService: JiraUpdatedInitiativePersistenceService,
    private val taskPersistenceService: JiraUpdatedTaskPersistenceService,
    private val jiraIssueKeyExtractor: JiraIssueKeyExtractor,
    private val jiraDateTimeParser: JiraDateTimeParser,
) {

    companion object {
        private const val CANCELLED_STATUS_NAME = "Отменена"
        private const val CLASSIC_ML_LABEL = "ClassicML"
        private val MINIMUM_UPDATE_AGE = Duration.ofHours(1)
        private val log by logger()
    }

    /**
     * Постранично ищет обновлённые инициативы.
     *
     * Окончательная ошибка Search прекращает запуск. Ошибка обработки
     * одной инициативы не мешает перейти к следующей.
     */
    fun updateInitiatives(
        updateDepth: Int,
        maxResults: Int,
        referenceData: JiraImportReferenceData,
        jiraErrorTracker: JiraErrorTracker,
        pultUpdatedBeforeTaskSync: Map<Long, LocalDateTime?>,
    ) {
        val processedJiraKeys = mutableSetOf<String>()
        var updatedInitiatives = 0
        var archivedInitiatives = 0

        log.info("Started searching updated Jira initiatives: updateDepth={}, maxResults={}", updateDepth, maxResults)

        jiraSearchPaginator.processPages(
            maxResults = maxResults,
            jiraErrorTracker = jiraErrorTracker,
            requestFactory = { startAt ->
                searchRequestFactory.createUpdatedInitiativesRequest(updateDepth, maxResults, startAt)
            },
            pageProcessor = { response ->
                if (response.total == 0) log.info("No updated initiatives found in Jira")

                val jiraKeys = response.issues.mapNotNull { jiraIssueKeyExtractor.extractCrossgoalKey(it.key) }.toSet()
                if (jiraKeys.isNotEmpty()) {
                    val referencesByJiraKey = initiativeRepository.findInitiativeReferences(jiraKeys)
                        .groupBy { reference -> reference.jiraKey }

                    response.issues.forEach { issue ->
                        val jiraKey = jiraIssueKeyExtractor.extractCrossgoalKey(issue.key)
                        if (jiraKey == null || !processedJiraKeys.add(jiraKey)) return@forEach

                        val agentIds = referencesByJiraKey[jiraKey].orEmpty().map { it.agentId }.distinct()
                        if (agentIds.isEmpty()) {
                            log.debug("Jira initiative is absent in Pult: jiraKey={}", jiraKey)
                            return@forEach
                        }

                        if (agentIds.size != 1) {
                            log.warn("Skipping initiative with ambiguous Jira key: jiraKey={}, agentIds={}", jiraKey, agentIds)
                            return@forEach
                        }

                        val agentId = agentIds.single()
                        try {
                            val cancelled = issue.fields?.status?.name.equals(CANCELLED_STATUS_NAME, ignoreCase = true)
                            val classicMl = issue.fields?.labels.orEmpty()
                                .any { label -> label.equals(CLASSIC_ML_LABEL, ignoreCase = true) }

                            if (cancelled || classicMl) {
                                taskPersistenceService.archiveInitiative(agentId, jiraKey)
                                archivedInitiatives++
                                return@forEach
                            }

                            val jiraCreated = jiraDateTimeParser.parse(issue.fields?.created)
                            val jiraUpdated = jiraDateTimeParser.parse(issue.fields?.updated)

                            if (jiraCreated == null || jiraUpdated == null) {
                                log.warn("Skipping initiative with missing or invalid Jira dates: jiraKey={}", jiraKey)
                                return@forEach
                            }

                            if (Duration.between(jiraCreated, jiraUpdated) < MINIMUM_UPDATE_AGE) {
                                log.debug("Skipping initiative updated within an hour of creation: jiraKey={}", jiraKey)
                                return@forEach
                            }

                            val result = initiativePersistenceService.updateInitiative(
                                agentId = agentId,
                                issue = issue,
                                jiraUpdated = jiraUpdated,
                                referenceData = referenceData,
                                pultUpdatedBeforeTaskSync = pultUpdatedBeforeTaskSync
                            )

                            if (result == JiraUpdatedInitiativeResult.UPDATED) updatedInitiatives++
                        } catch (exception: Exception) {
                            log.error("Failed to update Jira initiative: jiraKey={}, agentId={}", jiraKey, agentId, exception)
                            taskPersistenceService.markSynchronizationError(agentId)
                        }
                    }
                }
            }
        )

        log.info("Finished updating Jira initiatives: updated={}, archived={}", updatedInitiatives, archivedInitiatives)
    }
}

/**
 * Выполняет FR2: обновляет изменённые Task, пересчитывает статусы
 * затронутых инициатив и обрабатывает изменения самих инициатив.
 *
 * Jira HTTP-вызовы выполняются вне транзакций сохранения.
 */
@Service
class JiraInitiativeUpdateService(
    private val optionsService: OptionsService,
    private val referenceDataProvider: JiraImportReferenceDataProvider,
    private val searchRequestFactory: JiraUpdateSearchRequestFactory,
    private val jiraSearchPaginator: JiraSearchPaginator,
    private val jiraIssueRepository: JiraIssueRepository,
    private val jiraIssueKeyExtractor: JiraIssueKeyExtractor,
    private val jiraUpdateIssueReader: JiraUpdateIssueReader,
    private val taskPersistenceService: JiraUpdatedTaskPersistenceService,
    private val updatedStageStatusService: JiraUpdatedStageStatusService,
    private val updatedInitiativeService: JiraUpdatedInitiativeService,
) {

    companion object {
        private const val CANCELLED_STATUS_NAME = "Отменена"
        private const val CLASSIC_ML_LABEL = "ClassicML"
        private val INITIATIVE_FIELDS = listOf("status", "labels")
        private val log by logger()
    }

    /**
     * Выполняет один запуск FR2.
     *
     * Счётчик Jira-ошибок и данные обработки создаются на запуск.
     * Перед поиском обновлений инициатив счётчик сбрасывается согласно
     * документации второго подпроцесса.
     */
    fun synchronizeUpdates() {
        val jiraErrorTracker = JiraErrorTracker()
        val initiativeStates = mutableMapOf<Long, JiraUpdatedInitiativeState>()
        val processedTaskKeys = mutableSetOf<String>()
        val affectedAgentIds = mutableSetOf<Long>()
        val pultUpdatedBeforeTaskSync = mutableMapOf<Long, LocalDateTime?>()

        var receivedTasks = 0
        var existingTasks = 0
        var skippedTasks = 0
        var archivedInitiatives = 0

        log.info("Started FromJiraUpdate scheduler")

        try {
            val options = optionsService.getCurrent()
            val updateDepth = requireNotNull(options.updateDepth) { "Jira updateDepth is not configured" }
            val maxResults = requireNotNull(options.maxResults) { "Jira maxResults is not configured" }
            require(updateDepth > 0) { "Jira updateDepth must be positive" }
            require(maxResults > 0) { "Jira maxResults must be positive" }

            val referenceData = referenceDataProvider.load()
            log.info("Started searching updated monitoring Tasks: updateDepth={}, maxResults={}", updateDepth, maxResults)

            jiraSearchPaginator.processPages(
                maxResults = maxResults,
                jiraErrorTracker = jiraErrorTracker,
                requestFactory = { startAt ->
                    searchRequestFactory.createUpdatedTasksRequest(updateDepth, maxResults, startAt)
                },
                pageProcessor = { response ->
                    if (response.total == 0) log.info("No updated monitoring Tasks found in Jira")

                    val statistics = processUpdatedTaskPage(
                        tasks = response.issues,
                        initiativeStates = initiativeStates,
                        processedTaskKeys = processedTaskKeys,
                        jiraErrorTracker = jiraErrorTracker,
                        pultUpdatedBeforeTaskSync = pultUpdatedBeforeTaskSync
                    )

                    receivedTasks += statistics.receivedTasks
                    existingTasks += statistics.existingTasks
                    skippedTasks += statistics.skippedTasks
                    archivedInitiatives += statistics.archivedAgentIds.size
                    affectedAgentIds += statistics.affectedAgentIds
                    affectedAgentIds.removeAll(statistics.archivedAgentIds)
                }
            )

            updatedStageStatusService.updateStatuses(
                affectedAgentIds = affectedAgentIds,
                statusesByCode = referenceData.statusesByCode,
                jiraErrorTracker = jiraErrorTracker
            )

            log.info(
                "Finished updated Task processing: received={}, existing={}, skipped={}, archived={}, affectedAgents={}, jiraErrorCount={}",
                receivedTasks, existingTasks, skippedTasks, archivedInitiatives,
                affectedAgentIds.size, jiraErrorTracker.getErrorCount()
            )

            jiraErrorTracker.reset()

            updatedInitiativeService.updateInitiatives(
                updateDepth = updateDepth,
                maxResults = maxResults,
                referenceData = referenceData,
                jiraErrorTracker = jiraErrorTracker,
                pultUpdatedBeforeTaskSync = pultUpdatedBeforeTaskSync
            )
        } catch (exception: JiraErrorLimitExceededException) {
            log.error(
                "FromJiraUpdate stopped: Jira error limit exceeded, jiraErrorCount={}",
                jiraErrorTracker.getErrorCount(), exception
            )
        } catch (exception: Exception) {
            log.error(
                "FromJiraUpdate stopped due to an error: jiraErrorCount={}, error={}",
                jiraErrorTracker.getErrorCount(), exception.message, exception
            )
        } finally {
            log.info("Finished FromJiraUpdate scheduler: jiraErrorCount={}", jiraErrorTracker.getErrorCount())
        }
    }

    /**
     * Сопоставляет Task страницы с сохранёнными jira_issue.
     *
     * Task группируются по инициативе: статус самой инициативы
     * проверяется один раз за запуск, а QG/SLA сохраняются одной
     * транзакцией для каждой группы.
     */
    private fun processUpdatedTaskPage(
        tasks: List<SearchIssueDto>,
        initiativeStates: MutableMap<Long, JiraUpdatedInitiativeState>,
        processedTaskKeys: MutableSet<String>,
        jiraErrorTracker: JiraErrorTracker,
        pultUpdatedBeforeTaskSync: MutableMap<Long, LocalDateTime?>,
    ): JiraUpdatedTaskPageStatistics {
        if (tasks.isEmpty()) return JiraUpdatedTaskPageStatistics()

        val jiraKeys = tasks.mapNotNull { task -> jiraIssueKeyExtractor.extractCrossgoalKey(task.key) }.toSet()
        val savedTasksByJiraKey = if (jiraKeys.isEmpty()) {
            emptyMap()
        } else {
            jiraIssueRepository.findMonitoringTasksForUpdate(jiraKeys, JiraIssueType.task.name)
                .groupBy { savedTask -> jiraIssueKeyExtractor.extractCrossgoalKey(savedTask.jiraKey) }
        }

        val changesByAgentId = linkedMapOf<Long, MutableList<JiraUpdatedTaskChange>>()
        val initiativeKeysByAgentId = mutableMapOf<Long, String?>()
        var existingTasks = 0
        var skippedTasks = 0

        tasks.forEach { task ->
            val jiraKey = jiraIssueKeyExtractor.extractCrossgoalKey(task.key)
            val savedTasks = savedTasksByJiraKey[jiraKey].orEmpty()

            if (jiraKey == null || savedTasks.size != 1 || !processedTaskKeys.add(jiraKey)) {
                skippedTasks++

                if (savedTasks.size > 1) {
                    log.warn("Skipping Jira Task with ambiguous relations: jiraKey={}, relations={}", jiraKey, savedTasks.size)
                } else {
                    log.debug("Skipping absent or repeated Jira Task: jiraKey={}", task.key)
                }
                return@forEach
            }

            val savedTask = savedTasks.single()
            val agent = savedTask.agent
            val qualityGate = savedTask.qualityGate

            if (agent == null || agent.disabled == true || qualityGate == null) {
                skippedTasks++
                log.debug("Skipping Jira Task without active agent or quality gate: jiraKey={}", jiraKey)
                return@forEach
            }

            // Пересчёт статуса ниже изменит ai_agent.updated. Для следующего
            // подпроцесса нужно сохранить значение до изменений этого запуска.
            if (!pultUpdatedBeforeTaskSync.containsKey(agent.id)) {
                pultUpdatedBeforeTaskSync[agent.id] = agent.updated
            }

            existingTasks++
            changesByAgentId.getOrPut(agent.id) { mutableListOf() }
                .add(JiraUpdatedTaskChange(task = task, qualityGate = qualityGate))

            initiativeKeysByAgentId.putIfAbsent(
                agent.id,
                jiraIssueKeyExtractor.extractCrossgoalKey(agent.agentJiraUrl)
                    ?: jiraIssueKeyExtractor.extractCrossgoalKey(agent.agentId)
            )
        }

        val affectedAgentIds = mutableSetOf<Long>()
        val archivedAgentIds = mutableSetOf<Long>()

        changesByAgentId.forEach { (agentId, changes) ->
            try {
                val initiativeJiraKey = initiativeKeysByAgentId[agentId]
                val state = initiativeStates[agentId] ?: resolveInitiativeState(
                    agentId = agentId,
                    initiativeJiraKey = initiativeJiraKey,
                    jiraErrorTracker = jiraErrorTracker
                ).also { resolvedState -> initiativeStates[agentId] = resolvedState }

                when (state) {
                    JiraUpdatedInitiativeState.ARCHIVED -> {
                        taskPersistenceService.archiveInitiative(agentId, requireNotNull(initiativeJiraKey))
                        archivedAgentIds += agentId
                    }

                    JiraUpdatedInitiativeState.ACTIVE -> {
                        if (taskPersistenceService.saveUpdatedTasks(agentId, changes)) {
                            affectedAgentIds += agentId
                        }
                    }

                    JiraUpdatedInitiativeState.UNAVAILABLE -> {
                        log.warn("Skipping Task updates because Jira initiative status is unavailable: agentId={}", agentId)
                    }
                }
            } catch (exception: JiraErrorLimitExceededException) {
                throw exception
            } catch (exception: Exception) {
                log.error("Failed to process updated Tasks: agentId={}, error={}", agentId, exception.message, exception)
                taskPersistenceService.markSynchronizationError(agentId)
            }
        }

        return JiraUpdatedTaskPageStatistics(
            receivedTasks = tasks.size,
            existingTasks = existingTasks,
            skippedTasks = skippedTasks,
            affectedAgentIds = affectedAgentIds,
            archivedAgentIds = archivedAgentIds
        )
    }

    /**
     * Проверяет статус и метки родительской инициативы непосредственно в Jira.
     *
     * Отменённая инициатива и инициатива с ClassicML подлежат архивации.
     * При ошибке GET или отсутствии статуса Task не обновляются.
     */
    private fun resolveInitiativeState(
        agentId: Long,
        initiativeJiraKey: String?,
        jiraErrorTracker: JiraErrorTracker,
    ): JiraUpdatedInitiativeState {
        if (initiativeJiraKey == null) {
            log.warn("Cannot determine CROSSGOAL initiative key: agentId={}", agentId)
            taskPersistenceService.markSynchronizationError(agentId)
            return JiraUpdatedInitiativeState.UNAVAILABLE
        }

        val jiraInitiative = jiraUpdateIssueReader.getIssue(initiativeJiraKey, INITIATIVE_FIELDS, jiraErrorTracker)
            ?: run {
                taskPersistenceService.markSynchronizationError(agentId)
                return JiraUpdatedInitiativeState.UNAVAILABLE
            }

        val statusName = jiraInitiative.fields?.status?.name
        val hasClassicMlLabel = jiraInitiative.fields?.labels.orEmpty()
            .any { label -> label.equals(CLASSIC_ML_LABEL, ignoreCase = true) }

        if (statusName.equals(CANCELLED_STATUS_NAME, ignoreCase = true) || hasClassicMlLabel) {
            return JiraUpdatedInitiativeState.ARCHIVED
        }

        if (statusName.isNullOrBlank()) {
            log.warn("Jira initiative has no status: agentId={}, jiraKey={}", agentId, initiativeJiraKey)
            taskPersistenceService.markSynchronizationError(agentId)
            return JiraUpdatedInitiativeState.UNAVAILABLE
        }

        return JiraUpdatedInitiativeState.ACTIVE
    }
}




```
