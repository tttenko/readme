```java

/**
 * Завершает FR2-синхронизацию инициатив, которым не требуется
 * обработка monitoring Task.
 *
 * Для PULT-инициатив сохраняет текущий бизнес-статус.
 * Для инициатив с меткой AI-эффективность либо без monitoring Epic
 * устанавливает статус analysis.
 */
@Service
class JiraUpdatedInitiativeCompletionService(
    private val agentRepository: AIAgentRepository,
) {

    companion object {
        private const val ANALYSIS_STATUS_CODE = "analysis"
        private val log by logger()
    }

    /**
     * Завершает синхронизацию PULT-инициативы без изменения
     * её бизнес-статуса и данных мониторинга.
     *
     * Архивированная инициатива пропускается.
     */
    @Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = [Exception::class])
    fun completePultInitiative(agentId: Long, jiraKey: String) {
        val agent = agentRepository.findByIdForUpdate(agentId)
            ?: throw AiBadRequestException(
                errorCode = INITIATIVE_NOT_FOUND,
                message = "Initiative not found: agentId=$agentId"
            )

        if (agent.disabled == true) return

        completeSynchronization(agent)
        log.debug("PULT initiative synchronized without monitoring: agentId={}, jiraKey={}", agentId, jiraKey)
    }

    /**
     * Завершает синхронизацию инициативы с меткой AI-эффективность
     * либо без monitoring Epic.
     *
     * Устанавливает бизнес-статус analysis.
     * Архивированная инициатива пропускается.
     */
    @Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = [Exception::class])
    fun completeWithoutMonitoring(agentId: Long, jiraKey: String, statusesByCode: Map<String, StatusEntity>) {
        val analysisStatus = requireNotNull(statusesByCode[ANALYSIS_STATUS_CODE]) {
            "Active status '$ANALYSIS_STATUS_CODE' is not configured"
        }

        val agent = agentRepository.findByIdForUpdate(agentId)
            ?: throw AiBadRequestException(
                errorCode = INITIATIVE_NOT_FOUND,
                message = "Initiative not found: agentId=$agentId"
            )

        if (agent.disabled == true) return

        agent.agentStatus = analysisStatus
        completeSynchronization(agent)
        log.debug("Initiative synchronized without monitoring: agentId={}, jiraKey={}", agentId, jiraKey)
    }

    /** Записывает технический статус успешной синхронизации и обновляет даты. */
    private fun completeSynchronization(agent: AIAgentEntity) {
        val currentDateTime = LocalDateTime.now()
        agent.jiraFromStatus = "done"
        agent.jiraUpdated = currentDateTime
        agent.updated = currentDateTime
        agentRepository.save(agent)
    }
}

/**
 * Обновляет связи существующей инициативы по данным Jira.
 *
 * Стратегии и задействованные ресурсы обновляются для всех инициатив.
 * Enablers и GigaUsage обновляются только для инициатив,
 * идентификатор которых не содержит PULT.
 *
 * Контакты инициативы не изменяются.
 */
@Service
class JiraUpdatedInitiativeRelationsService(
    private val agentRepository: AIAgentRepository,
    private val agentStrategyRepository: AgentStrategyRepository,
    private val involvedResourceRepository: InvolvedResourceRepository,
    private val enablerRepository: EnablerRepository,
    private val jiraIssueRepository: JiraIssueRepository,
    private val jiraService: JiraService,
    private val strategyResolver: JiraStrategyResolver,
    private val involvedResourceResolver: JiraInvolvedResourceResolver,
    private val gigaUsageIssueResolver: JiraGigaUsageIssueResolver,
    private val enablerNameNormalizer: EnablerNameNormalizer,
) {

    companion object {
        private const val GIGAUSAGE_PROJECT = "gigausage"
        private val log by logger()
    }

    /**
     * Обновляет связанные сущности одной инициативы в отдельной транзакции.
     *
     * @return true, если инициатива создана в Пульте и дальнейшая
     * обработка monitoring не требуется.
     */
    @Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = [Exception::class])
    fun updateRelations(agentId: Long, issue: SearchIssueDto, referenceData: JiraImportReferenceData): Boolean {
        val agent = agentRepository.findByIdForUpdate(agentId)
            ?: throw AiBadRequestException(
                errorCode = INITIATIVE_NOT_FOUND,
                message = "Initiative not found: agentId=$agentId"
            )

        val jiraKey = requireNotNull(issue.key) { "Jira initiative key is missing: agentId=$agentId" }
        updateStrategies(agent, issue, referenceData)
        updateInvolvedResources(agent, issue, jiraKey)

        val createdInPult = agent.agentId?.contains("PULT", ignoreCase = true) == true
        if (createdInPult) {
            log.debug("Skipped Jira enablers and GigaUsage for PULT initiative: agentId={}, jiraKey={}", agentId, jiraKey)
            return true
        }

        updateEnablers(agent, issue, referenceData)
        updateGigaUsageIssues(agent, issue)
        return false
    }

    /**
     * Приводит связи со стратегиями к актуальному набору Jira.
     *
     * Удаляет отсутствующие в Jira связи, сохраняет существующие
     * и добавляет новые. Для актуальных связей устанавливает jiraLink = done.
     */
    private fun updateStrategies(
        agent: AIAgentEntity,
        issue: SearchIssueDto,
        referenceData: JiraImportReferenceData,
    ) {
        val strategies = strategyResolver.resolveStrategies(issue, referenceData.strategiesByJiraKey)
        val strategyIds = strategies.map { strategy -> strategy.id }.toSet()
        val existingStrategies = agentStrategyRepository.findAllByAgentId(agent.id)

        val obsoleteStrategies = existingStrategies.filter { relation -> relation.strategy?.id !in strategyIds }
        if (obsoleteStrategies.isNotEmpty()) agentStrategyRepository.deleteAll(obsoleteStrategies)

        val existingByStrategyId = existingStrategies.associateBy { relation -> relation.strategy?.id }
        val updatedStrategies = strategies.map { strategy ->
            (existingByStrategyId[strategy.id] ?: AgentStrategyEntity(agent = agent, strategy = strategy))
                .apply { jiraLink = "done" }
        }

        if (updatedStrategies.isNotEmpty()) agentStrategyRepository.saveAll(updatedStrategies)
        log.debug("Updated initiative strategies: agentId={}, strategyCount={}", agent.id, updatedStrategies.size)
    }

    /**
     * Обновляет задействованные ресурсы по данным Jira.
     *
     * Сопоставляет записи по составному идентификатору:
     * инициатива, источник и тип ресурса.
     */
    private fun updateInvolvedResources(agent: AIAgentEntity, issue: SearchIssueDto, jiraKey: String) {
        val resources = involvedResourceResolver.resolveInvolvedResources(jiraKey, issue)
        val currentResources = involvedResourceRepository.findAllByAiAgentId(agent.id)
        val currentById = currentResources.associateBy { resource -> resource.id }
        val currentDateTime = LocalDateTime.now()

        val updatedResources = resources.map { resource ->
            val resourceId = InvolvedResourceEmbeddedId(
                aiAgentId = agent.id,
                source = resource.source,
                type = resource.type
            )

            (currentById[resourceId] ?: InvolvedResourceEntity().apply {
                id = resourceId
                aiAgent = agent
                created = currentDateTime
            }).apply {
                value = resource.value
                timeAllocated = null
                updated = currentDateTime
            }
        }

        val updatedIds = updatedResources.map { resource -> resource.id }.toSet()
        val obsoleteResources = currentResources.filter { resource -> resource.id !in updatedIds }

        if (obsoleteResources.isNotEmpty()) involvedResourceRepository.deleteAll(obsoleteResources)
        if (updatedResources.isNotEmpty()) involvedResourceRepository.saveAll(updatedResources)

        log.debug("Updated initiative resources: agentId={}, resourceCount={}", agent.id, updatedResources.size)
    }

    /**
     * Заменяет связи с энейблерами выбранными в Jira значениями.
     *
     * Учитывает только элементы с checked = true.
     * Имена сопоставляются со справочником после нормализации.
     */
    private fun updateEnablers(agent: AIAgentEntity, issue: SearchIssueDto, referenceData: JiraImportReferenceData) {
        val enablerIds = issue.fields?.customfield_15903.orEmpty()
            .filter { option -> option.checked == true }
            .mapNotNull { option ->
                val normalizedName = enablerNameNormalizer.normalize(option.name)
                val enabler = normalizedName?.let(referenceData.enablersByNormalizedName::get)

                if (enabler == null) {
                    log.warn(
                        "Jira enabler was not found in Pult: agentId={}, jiraKey={}, name={}",
                        agent.id, issue.key, option.name
                    )
                }

                enabler?.id
            }
            .distinct()

        enablerRepository.deleteAllByAgentId(agent.id)
        if (enablerIds.isNotEmpty()) enablerRepository.addAllToAgent(agent.id, enablerIds)

        log.debug("Updated initiative enablers: agentId={}, enablerCount={}", agent.id, enablerIds.size)
    }

    /**
     * Обновляет все GigaUsage-связи инициативы.
     *
     * Существующие записи сопоставляются по Jira key.
     * Связи, отсутствующие в актуальных данных Jira, удаляются.
     */
    private fun updateGigaUsageIssues(agent: AIAgentEntity, issue: SearchIssueDto) {
        val gigaUsageIssues = gigaUsageIssueResolver.resolveGigaUsageIssues(issue)
        val existingIssues = jiraIssueRepository.findByAgentIdAndTypeAndProject(
            agent.id, JiraIssueType.initiative.name, GIGAUSAGE_PROJECT
        )
        val existingByKey = existingIssues.associateBy { jiraIssue -> jiraIssue.jiraKey?.uppercase() }
        val currentDateTime = LocalDateTime.now()

        val updatedIssues = gigaUsageIssues.distinctBy { gigaUsageIssue -> gigaUsageIssue.jiraKey.uppercase() }
            .map { gigaUsageIssue ->
                val jiraKey = gigaUsageIssue.jiraKey
                val jiraIssue = existingByKey[jiraKey.uppercase()] ?: JiraIssueEntity(
                    agent = agent,
                    type = JiraIssueType.initiative.name,
                    project = GIGAUSAGE_PROJECT,
                    jiraKey = jiraKey
                ).apply {
                    created = currentDateTime
                }

                jiraIssue.apply {
                    jiraId = gigaUsageIssue.jiraId
                    jiraUrl = jiraService.getJiraSigmaUrl() + jiraKey
                }
            }

        val updatedKeys = updatedIssues.mapNotNull { jiraIssue -> jiraIssue.jiraKey?.uppercase() }.toSet()
        val obsoleteIssues = existingIssues.filter { jiraIssue -> jiraIssue.jiraKey?.uppercase() !in updatedKeys }

        if (obsoleteIssues.isNotEmpty()) jiraIssueRepository.deleteAll(obsoleteIssues)
        if (updatedIssues.isNotEmpty()) jiraIssueRepository.saveAll(updatedIssues)

        log.debug("Updated GigaUsage relations: agentId={}, issueCount={}", agent.id, updatedIssues.size)
    }
}

/**
 * Удаляет устаревшие связи с Task monitoring Epic.
 *
 * Вызывается только после успешного получения всех страниц Jira Search
 * и сохранения актуальных Task.
 */
@Service
class JiraUpdatedMonitoringTaskCleanupService(
    private val jiraIssueRepository: JiraIssueRepository,
) {

    private val log by logger()

    /** Оставляет у Epic только Task из актуального сопоставленного набора. */
    @Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = [Exception::class])
    fun deleteObsoleteTasks(agentId: Long, epicIssueId: Long, taskMatches: List<JiraTaskQualityGateMatch>) {
        val currentTaskKeys = taskMatches.mapNotNull { match -> match.task.key?.uppercase() }.toSet()
        val savedTasks = jiraIssueRepository.findAllByParentIdAndType(epicIssueId, JiraIssueType.task.name)

        val obsoleteTasks = savedTasks.filter { task ->
            task.agent?.id == agentId &&
                task.project?.equals("crossgoal", ignoreCase = true) == true &&
                task.jiraKey?.uppercase()?.let(currentTaskKeys::contains) != true
        }

        if (obsoleteTasks.isEmpty()) return

        jiraIssueRepository.deleteAll(obsoleteTasks)
        log.info(
            "Deleted obsolete monitoring Tasks: agentId={}, epicIssueId={}, count={}",
            agentId, epicIssueId, obsoleteTasks.size
        )
    }
}

/**
 * Синхронизирует monitoring Epic и Task существующей Jira-инициативы.
 *
 * Использует существующие компоненты FR1 для поиска monitoring,
 * сопоставления Task с quality gate и сохранения результата.
 *
 * Jira HTTP-вызовы выполняются вне транзакций сохранения.
 */
@Service
class JiraUpdatedInitiativeMonitoringService(
    private val monitoringEpicResolver: JiraMonitoringEpicResolver,
    private val monitoringTaskSearchService: JiraMonitoringTaskSearchService,
    private val taskQualityGateMatcher: JiraTaskQualityGateMatcher,
    private val monitoringPersistenceService: JiraMonitoringPersistenceService,
    private val completionService: JiraUpdatedInitiativeCompletionService,
    private val monitoringTaskCleanupService: JiraUpdatedMonitoringTaskCleanupService,
) {

    private val log by logger()

    /**
     * Обрабатывает monitoring обновлённой инициативы.
     *
     * Для AI-эффективность и при отсутствии monitoring Epic
     * завершает синхронизацию со статусом analysis без поиска Task.
     *
     * Если Epic найден, получает полный набор Task с пагинацией,
     * проверяет наличие этапов и сохраняет monitoring-результат.
     */
    fun synchronizeMonitoring(
        agentId: Long,
        issue: SearchIssueDto,
        referenceData: JiraImportReferenceData,
        maxResults: Int,
        jiraErrorTracker: JiraErrorTracker,
    ) {
        val jiraKey = requireNotNull(issue.key) { "Jira initiative key is missing: agentId=$agentId" }

        if (!monitoringEpicResolver.isMonitoringRequired(issue.fields?.labels)) {
            completionService.completeWithoutMonitoring(agentId, jiraKey, referenceData.statusesByCode)
            log.debug("Skipped monitoring for AI-effectiveness initiative: agentId={}, jiraKey={}", agentId, jiraKey)
            return
        }

        val monitoringEpic = monitoringEpicResolver.findMonitoringEpic(jiraKey, issue.fields?.issuelinks)
        if (monitoringEpic == null) {
            completionService.completeWithoutMonitoring(agentId, jiraKey, referenceData.statusesByCode)
            log.debug("Monitoring Epic was not found: agentId={}, jiraKey={}", agentId, jiraKey)
            return
        }

        val epicIssueId = monitoringPersistenceService.saveMonitoringEpic(agentId, monitoringEpic)
        val monitoringTasks = monitoringTaskSearchService.searchMonitoringTasks(
            epicKey = monitoringEpic.jiraKey,
            maxResults = maxResults,
            jiraErrorTracker = jiraErrorTracker
        )
        val taskMatches = taskQualityGateMatcher.matchTasks(
            initiativeJiraKey = jiraKey,
            tasks = monitoringTasks,
            qualityGates = referenceData.qualityGates
        )

        val hasValidStage = taskMatches.any { match ->
            match.qualityGate.type == QualityGateType.status &&
                match.qualityGate.status?.code != null &&
                !match.task.fields?.status?.id.isNullOrBlank()
        }

        if (!hasValidStage) {
            throw AiBadRequestException(
                errorCode = JIRA_SYNC_ERROR,
                message = "No valid monitoring stage Tasks: agentId=$agentId, jiraKey=$jiraKey, epicKey=${monitoringEpic.jiraKey}"
            )
        }

        monitoringPersistenceService.saveMonitoringData(
            agentId = agentId,
            initiativeJiraKey = jiraKey,
            monitoringEpicIssueId = epicIssueId,
            monitoringEpicKey = monitoringEpic.jiraKey,
            taskMatches = taskMatches,
            referenceData = referenceData
        )

        monitoringTaskCleanupService.deleteObsoleteTasks(agentId, epicIssueId, taskMatches)

        log.debug(
            "Updated initiative monitoring: agentId={}, jiraKey={}, epicKey={}, tasks={}, matchedTasks={}",
            agentId, jiraKey, monitoringEpic.jiraKey, monitoringTasks.size, taskMatches.size
        )
    }
}

/**
 * Ищет обновлённые Jira-инициативы и выполняет этапы 3 и 4 FR2.
 *
 * Сначала обрабатывает архивацию, затем проверяет даты,
 * обновляет разрешённые поля инициативы и её связи.
 *
 * PULT-инициативы завершаются без обработки monitoring.
 * Для остальных инициатив применяется отдельная ветка monitoring,
 * учитывающая метку AI-эффективность и наличие monitoring Epic.
 */
@Service
class JiraUpdatedInitiativeProcessor(
    private val searchRequestFactory: JiraUpdateSearchRequestFactory,
    private val jiraSearchPaginator: JiraSearchPaginator,
    private val initiativeRepository: JiraUpdatedInitiativeRepository,
    private val initiativePersistenceService: JiraUpdatedInitiativePersistenceService,
    private val initiativeRelationsService: JiraUpdatedInitiativeRelationsService,
    private val initiativeMonitoringService: JiraUpdatedInitiativeMonitoringService,
    private val initiativeCompletionService: JiraUpdatedInitiativeCompletionService,
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
     * Постранично обрабатывает изменения существующих Jira-инициатив.
     *
     * Инициативы со статусом Отменена или меткой ClassicML архивируются
     * до проверки дат создания и изменения.
     *
     * Ошибка отдельной инициативы устанавливает технический статус error
     * и позволяет продолжить обработку. Превышение лимита Jira-ошибок
     * передаётся вызывающему сервису для остановки запуска.
     *
     * @param pultUpdatedBeforeTaskSync даты изменения инициатив до обработки
     * Task в текущем запуске; используются при сравнении дат Jira и Пульта.
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

                val jiraKeys = response.issues.mapNotNull { issue ->
                    jiraIssueKeyExtractor.extractCrossgoalKey(issue.key)
                }.toSet()

                if (jiraKeys.isNotEmpty()) {
                    val referencesByJiraKey = initiativeRepository.findInitiativeReferences(jiraKeys)
                        .groupBy { reference -> reference.jiraKey }

                    response.issues.forEach { issue ->
                        val jiraKey = jiraIssueKeyExtractor.extractCrossgoalKey(issue.key)
                        if (jiraKey == null || !processedJiraKeys.add(jiraKey)) return@forEach

                        val agentIds = referencesByJiraKey[jiraKey].orEmpty()
                            .map { reference -> reference.agentId }
                            .distinct()

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
                            if (result != JiraUpdatedInitiativeResult.UPDATED) return@forEach

                            val createdInPult = initiativeRelationsService.updateRelations(agentId, issue, referenceData)

                            if (createdInPult) {
                                initiativeCompletionService.completePultInitiative(agentId, jiraKey)
                            } else {
                                initiativeMonitoringService.synchronizeMonitoring(
                                    agentId = agentId,
                                    issue = issue,
                                    referenceData = referenceData,
                                    maxResults = maxResults,
                                    jiraErrorTracker = jiraErrorTracker
                                )
                            }

                            updatedInitiatives++
                        } catch (exception: JiraErrorLimitExceededException) {
                            taskPersistenceService.markSynchronizationError(agentId)
                            throw exception
                        } catch (exception: Exception) {
                            log.error("Failed to synchronize Jira initiative: jiraKey={}, agentId={}", jiraKey, agentId, exception)
                            taskPersistenceService.markSynchronizationError(agentId)
                        }
                    }
                }
            }
        )

        log.info("Finished updating Jira initiatives: updated={}, archived={}", updatedInitiatives, archivedInitiatives)
    }
}




```
