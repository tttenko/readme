```java

/**
 * Одним запросом загружает сохранённые Task текущей страницы Jira Search.
 *
 * Вместе с Task загружает инициативу, quality gate и связанный статус:
 * обработка страницы не должна обращаться к ленивым связям по одной Task.
 */
@Query(
    """
    select distinct issue
    from JiraIssueEntity issue
    join fetch issue.agent agent
    left join fetch issue.qualityGate qualityGate
    left join fetch qualityGate.status
    where issue.jiraKey in :jiraKeys
      and issue.type = :taskType
      and lower(issue.project) = 'crossgoal'
    """
)
fun findMonitoringTasksForUpdate(
    @Param("jiraKeys") jiraKeys: Collection<String>,
    @Param("taskType") taskType: String,
): List<JiraIssueEntity>

/**
 * Загружает все сохранённые этапы инициативы перед повторным GET из Jira.
 *
 * Используются только Task, связанные с quality gate типа status.
 * Jira key и справочник этапа необходимы для расчёта статуса по алгоритму FR1.
 */
@Query(
    """
    select distinct issue
    from JiraIssueEntity issue
    join fetch issue.qualityGate qualityGate
    left join fetch qualityGate.status
    where issue.agent.id = :agentId
      and issue.type = :taskType
      and lower(issue.project) = 'crossgoal'
      and qualityGate.type = :stageType
    """
)
fun findStageTasksForUpdate(
    @Param("agentId") agentId: Long,
    @Param("taskType") taskType: String,
    @Param("stageType") stageType: QualityGateType,
): List<JiraIssueEntity>

/**
 * Изменённая Jira Task и её уже установленная связь со справочником quality gate.
 *
 * Сопоставление берётся из jira_issue: повторно определять quality gate
 * по названию Task не требуется.
 */
data class JiraUpdatedTaskChange(
    val task: SearchIssueDto,
    val qualityGate: QualityGateEntity,
)

/**
 * Статистика обработки одной страницы поиска Task.
 *
 * affectedAgentIds содержат инициативы с изменённым этапом.
 * archivedAgentIds позволяют исключить архивированные инициативы
 * из последующего пересчёта статуса.
 */
data class JiraUpdatedTaskPageStatistics(
    val receivedTasks: Int = 0,
    val existingTasks: Int = 0,
    val skippedTasks: Int = 0,
    val affectedAgentIds: Set<Long> = emptySet(),
    val archivedAgentIds: Set<Long> = emptySet(),
)

enum class JiraUpdatedInitiativeState {
    ACTIVE,
    ARCHIVED,
    UNAVAILABLE,
}

/**
 * Получает инициативы и этапы из Jira для FR2.
 *
 * Повторные попытки выполняются существующим @Retryable у Jira Feign client.
 * Счётчик текущего запуска увеличивается только тогда, когда вызов
 * завершился исключением после всех настроенных попыток.
 *
 * null означает окончательную ошибку конкретного GET, при которой
 * обработку соответствующей инициативы продолжать нельзя.
 */
@Component
class JiraUpdateIssueReader(
    private val jiraService: JiraService,
) {

    private val log by logger()

    /**
     * Получает issue по CROSSGOAL key.
     *
     * После окончательной ошибки учитывает её в общем счётчике FR2.
     * При превышении лимита выбрасывает исключение для остановки scheduler.
     */
    fun getIssue(issueKey: String, fields: List<String>, jiraErrorTracker: JiraErrorTracker): IssueDto? {
        return try {
            jiraService.getIssue(issueKey, fields)
        } catch (exception: Exception) {
            val jiraErrorCount = jiraErrorTracker.increment()
            log.error("Jira GET failed after retries: issueKey={}, jiraErrorCount={}, error={}",
                issueKey, jiraErrorCount, exception.message, exception)

            if (jiraErrorTracker.isErrorLimitExceeded()) throw JiraErrorLimitExceededException(exception)
            null
        }
    }
}

/**
 * Сохраняет изменения этапа 2 FR2 в базе данных.
 *
 * Jira HTTP-вызовов здесь нет. Изменения Task одной инициативы,
 * архивация и запись итогового статуса выполняются отдельными транзакциями.
 * Существующие правила обновления quality gate и SLA повторяют FR1.
 */
@Service
class JiraUpdatedTaskPersistenceService(
    private val agentRepository: AIAgentRepository,
    private val agentQualityGateRepository: AIAgentQualityGateRepository,
    private val agentStatusSlaRepository: AgentStatusSlaRepository,
    private val jiraDateTimeParser: JiraDateTimeParser,
) {

    companion object {
        private val COMPLETED_TASK_STATUS_IDS = setOf("10110", "5", "14103")
        private val log by logger()
    }

    /**
     * Обновляет QG и SLA по изменённым Task одной инициативы.
     *
     * Возвращает true, если среди обработанных Task был корректно связанный
     * этап типа status. Такая инициатива должна пройти повторный GET всех
     * сохранённых этапов и пересчёт текущего статуса.
     *
     * Существующие SLA загружаются одним запросом. Некорректная непустая
     * дата из Jira логируется и не затирает ранее сохранённую дату.
     */
    @Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = [Exception::class])
    fun saveUpdatedTasks(agentId: Long, changes: List<JiraUpdatedTaskChange>): Boolean {
        if (changes.isEmpty()) return false

        val agent = agentRepository.findByIdForUpdate(agentId) ?: return false
        if (agent.disabled == true) return false

        val currentDateTime = LocalDateTime.now()
        updateQualityGates(agentId, changes, currentDateTime)
        return updateStageSla(agent, changes)
    }

    /**
     * Переносит отменённую в Jira инициативу в архив Пульта.
     *
     * Повторный вызов безопасен: уже архивированная инициатива остаётся
     * без изменений. Метод не удаляет связанные записи и не отправляет
     * запросов в Jira.
     */
    @Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = [Exception::class])
    fun archiveInitiative(agentId: Long, initiativeJiraKey: String) {
        val agent = agentRepository.findByIdForUpdate(agentId) ?: return
        if (agent.disabled == true) return

        agent.disabled = true
        agent.updated = LocalDateTime.now()
        agentRepository.save(agent)

        log.info("Archived initiative from Jira: agentId={}, jiraKey={}", agentId, initiativeJiraKey)
    }

    /**
     * Сохраняет статус, рассчитанный по повторно полученным этапам.
     *
     * Если инициатива успела попасть в архив, статус больше не меняется.
     * Неизвестный код статуса считается ошибкой, чтобы не сохранить
     * частичный результат синхронизации.
     */
    @Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = [Exception::class])
    fun saveInitiativeStatus(agentId: Long, statusCode: String, statusesByCode: Map<String, StatusEntity>) {
        val status = requireNotNull(statusesByCode[statusCode]) {
            "Calculated Jira status '$statusCode' is missing from active status dictionary"
        }

        val agent = agentRepository.findByIdForUpdate(agentId) ?: return
        if (agent.disabled == true) return

        val currentDateTime = LocalDateTime.now()
        agent.agentStatus = status
        agent.jiraFromStatus = "done"
        agent.jiraUpdated = currentDateTime
        agent.updated = currentDateTime
        agentRepository.save(agent)

        log.info("Updated initiative status from Jira stages: agentId={}, statusCode={}", agentId, statusCode)
    }

    /**
     * Фиксирует неуспешную синхронизацию инициативы.
     *
     * Бизнес-статус agent_status_id сохраняется прежним. Архивированную
     * инициативу метод не изменяет.
     */
    @Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = [Exception::class])
    fun markSynchronizationError(agentId: Long) {
        val agent = agentRepository.findByIdForUpdate(agentId) ?: return
        if (agent.disabled == true) return

        val currentDateTime = LocalDateTime.now()
        agent.jiraFromStatus = "error"
        agent.jiraUpdated = currentDateTime
        agent.updated = currentDateTime
        agentRepository.save(agent)
    }

    /** Обновляет состояния вех, используя те же Jira status id, что и FR1. */
    private fun updateQualityGates(agentId: Long, changes: List<JiraUpdatedTaskChange>, currentDateTime: LocalDateTime) {
        val qualityGateChanges = changes.filter { change -> change.qualityGate.type == QualityGateType.quality_gate }

        qualityGateChanges.filter { change -> change.task.fields?.status?.id.isNullOrBlank() }
            .forEach { change -> log.warn("Cannot update QG without Jira Task status: agentId={}, taskKey={}", agentId, change.task.key) }

        val validChanges = qualityGateChanges.filter { change -> !change.task.fields?.status?.id.isNullOrBlank() }
        val checkedCodes = validChanges.filter { change -> change.task.fields?.status?.id in COMPLETED_TASK_STATUS_IDS }
            .mapNotNull { change -> change.qualityGate.code }.toSet()
        val uncheckedCodes = validChanges.filter { change -> change.task.fields?.status?.id !in COMPLETED_TASK_STATUS_IDS }
            .mapNotNull { change -> change.qualityGate.code }.filterNot(checkedCodes::contains).toSet()

        if (checkedCodes.isNotEmpty()) {
            agentQualityGateRepository.upsertStateForAgent(
                agentId = agentId, qualityGateCodes = checkedCodes,
                state = QualityGateState.checked.name, updatedAt = currentDateTime
            )
        }

        if (uncheckedCodes.isNotEmpty()) {
            agentQualityGateRepository.upsertStateForAgent(
                agentId = agentId, qualityGateCodes = uncheckedCodes,
                state = QualityGateState.unchecked.name, updatedAt = currentDateTime
            )
        }
    }

    /**
     * Обновляет сроки этапов из Jira Search.
     *
     * Пустое поле Jira очищает дату; некорректная непустая дата оставляет
     * предыдущее значение. Если этап не связан со статусом справочника,
     * он пропускается и не считается основанием для пересчёта статуса.
     */
    private fun updateStageSla(agent: AIAgentEntity, changes: List<JiraUpdatedTaskChange>): Boolean {
        val stageChanges = changes.filter { change -> change.qualityGate.type == QualityGateType.status }
        if (stageChanges.isEmpty()) return false

        val existingSlaByStatusId = agentStatusSlaRepository.findAllByAiAgentId(agent.id)
            .associateBy { sla -> sla.primaryKey.agentStatusId }.toMutableMap()
        val changedSlaByStatusId = linkedMapOf<Long, AgentStatusSlaEntity>()

        stageChanges.forEach { change ->
            val status = change.qualityGate.status
            val statusId = status?.id

            if (statusId == null) {
                log.warn("Stage Task has no linked status: agentId={}, taskKey={}, qualityGate={}",
                    agent.id, change.task.key, change.qualityGate.code)
                return@forEach
            }

            val sla = changedSlaByStatusId[statusId] ?: existingSlaByStatusId[statusId]
                ?: AgentStatusSlaEntity().apply {
                    aiAgent = agent
                    agentStatus = status
                }

            sla.plannedDate = parseDateOrKeepPrevious(
                value = change.task.fields?.customfield_16701,
                previousDate = sla.plannedDate,
                agentId = agent.id,
                taskKey = change.task.key,
                fieldName = "customfield_16701"
            )
            sla.completedDate = parseDateOrKeepPrevious(
                value = change.task.fields?.resolutiondate,
                previousDate = sla.completedDate,
                agentId = agent.id,
                taskKey = change.task.key,
                fieldName = "resolutiondate"
            )

            changedSlaByStatusId[statusId] = sla
        }

        if (changedSlaByStatusId.isNotEmpty()) agentStatusSlaRepository.saveAll(changedSlaByStatusId.values)
        return changedSlaByStatusId.isNotEmpty()
    }

    /** Парсит дату Jira и защищает ранее сохранённую дату от ошибочного формата. */
    private fun parseDateOrKeepPrevious(
        value: String?,
        previousDate: LocalDateTime?,
        agentId: Long,
        taskKey: String?,
        fieldName: String,
    ): LocalDateTime? {
        if (value.isNullOrBlank()) return null

        return jiraDateTimeParser.parse(value) ?: run {
            log.warn("Cannot parse Jira date; previous value retained: agentId={}, taskKey={}, field={}, value={}",
                agentId, taskKey, fieldName, value)
            previousDate
        }
    }
}

/**
 * Пересчитывает статусы инициатив, у которых изменились Jira Task этапов.
 *
 * Для каждой инициативы заново получает через GET все сохранённые этапы.
 * Статус вычисляется существующим резолвером FR1 только после успешного
 * получения каждого этапа. Ошибка одного GET не приводит к частичному
 * пересчёту и не останавливает обработку остальных инициатив, пока
 * общий лимит Jira-ошибок не превышен.
 */
@Service
class JiraUpdatedStageStatusService(
    private val jiraIssueRepository: JiraIssueRepository,
    private val jiraUpdateIssueReader: JiraUpdateIssueReader,
    private val taskPersistenceService: JiraUpdatedTaskPersistenceService,
    private val initiativeStatusResolver: JiraInitiativeStatusResolver,
    private val jiraIssueKeyExtractor: JiraIssueKeyExtractor,
) {

    companion object {
        private val STAGE_FIELDS = listOf(
            "summary", "issuetype", "description", "status", "customfield_16700",
            "customfield_16701", "lastViewed", "resolutiondate", "created", "updated"
        )
        private val log by logger()
    }

    /**
     * Последовательно пересчитывает статусы затронутых инициатив.
     *
     * Справочник статусов загружен один раз в основном сервисе FR2.
     * При достижении лимита ошибок передаёт исключение наверх, чтобы
     * текущий запуск scheduler завершился.
     */
    fun updateStatuses(
        affectedAgentIds: Set<Long>,
        statusesByCode: Map<String, StatusEntity>,
        jiraErrorTracker: JiraErrorTracker,
    ) {
        affectedAgentIds.forEach { agentId ->
            try {
                updateStatus(agentId, statusesByCode, jiraErrorTracker)
            } catch (exception: JiraErrorLimitExceededException) {
                taskPersistenceService.markSynchronizationError(agentId)
                throw exception
            } catch (exception: Exception) {
                log.error("Failed to update initiative status: agentId={}, error={}", agentId, exception.message, exception)
                taskPersistenceService.markSynchronizationError(agentId)
            }
        }
    }

    /**
     * Выполняет GET каждого этапа и сохраняет итоговый статус.
     *
     * Пустой набор этапов либо некорректная сохранённая связь считается
     * ошибкой синхронизации: без полного набора этапов нельзя безопасно
     * присвоить инициативе targetSolution.
     */
    private fun updateStatus(
        agentId: Long,
        statusesByCode: Map<String, StatusEntity>,
        jiraErrorTracker: JiraErrorTracker,
    ) {
        val savedStages = jiraIssueRepository.findStageTasksForUpdate(
            agentId = agentId,
            taskType = JiraIssueType.task.name,
            stageType = QualityGateType.status
        )

        if (savedStages.isEmpty()) {
            log.warn("Cannot calculate status without saved stage Tasks: agentId={}", agentId)
            taskPersistenceService.markSynchronizationError(agentId)
            return
        }

        val stageKeys = savedStages.mapNotNull { stage -> jiraIssueKeyExtractor.extractCrossgoalKey(stage.jiraKey) }

        if (stageKeys.size != savedStages.size || stageKeys.toSet().size != savedStages.size) {
            log.warn("Cannot calculate status with missing or duplicate stage keys: agentId={}", agentId)
            taskPersistenceService.markSynchronizationError(agentId)
            return
        }

        val stageMatches = mutableListOf<JiraTaskQualityGateMatch>()

        savedStages.forEach { savedStage ->
            val stageKey = requireNotNull(jiraIssueKeyExtractor.extractCrossgoalKey(savedStage.jiraKey))
            val jiraStage = jiraUpdateIssueReader.getIssue(stageKey, STAGE_FIELDS, jiraErrorTracker)

            if (jiraStage == null) {
                taskPersistenceService.markSynchronizationError(agentId)
                return
            }

            val jiraStatusId = jiraStage.fields?.status?.id
            if (jiraStatusId.isNullOrBlank()) {
                log.warn("Jira stage Task has no status: agentId={}, taskKey={}", agentId, stageKey)
                taskPersistenceService.markSynchronizationError(agentId)
                return
            }

            stageMatches += JiraTaskQualityGateMatch(
                task = SearchIssueDto(
                    id = savedStage.jiraId ?: stageKey,
                    key = stageKey,
                    fields = SearchIssueFieldsDto(
                        status = SearchIssueStatusDto(id = jiraStatusId, name = jiraStage.fields?.status?.name ?: "")
                    )
                ),
                qualityGate = requireNotNull(savedStage.qualityGate)
            )
        }

        val statusCode = initiativeStatusResolver.resolveStatusCode(stageMatches)
        taskPersistenceService.saveInitiativeStatus(agentId, statusCode, statusesByCode)
    }
}

/**
 * Оркестрирует первый подпроцесс FR2: поиск и обработку изменённых Task.
 *
 * Обрабатывает Jira Search постранично, проверяет наличие Task в Пульте
 * одним запросом на страницу, один раз за запуск проверяет Jira-инициативу
 * каждого agentId и сохраняет QG/SLA короткими транзакциями.
 *
 * После всех страниц повторно запрашивает этапы инициатив, затронутых
 * изменением SLA, и пересчитывает их статус через резолвер FR1.
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
) {

    companion object {
        private const val CANCELLED_STATUS_NAME = "Отменена"
        private const val CLASSIC_ML_LABEL = "ClassicML"
        private val INITIATIVE_FIELDS = listOf("status", "labels")
        private val log by logger()
    }

    /**
     * Выполняет один запуск обработки обновлений Jira.
     *
     * JiraErrorTracker и кеш состояния инициатив создаются на запуск:
     * данные разных ручных и плановых запусков не смешиваются.
     * Окончательная ошибка Jira Search прекращает подпроцесс;
     * ошибка GET отдельного этапа пропускает соответствующую инициативу.
     */
    fun synchronizeUpdates() {
        val jiraErrorTracker = JiraErrorTracker()
        val initiativeStates = mutableMapOf<Long, JiraUpdatedInitiativeState>()
        val processedTaskKeys = mutableSetOf<String>()
        val affectedAgentIds = mutableSetOf<Long>()

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
                        jiraErrorTracker = jiraErrorTracker
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

            // Этап 3: после обработки Task здесь начнётся поиск обновлённых инициатив.
        } catch (exception: JiraErrorLimitExceededException) {
            log.error("FromJiraUpdate stopped: Jira error limit exceeded, jiraErrorCount={}",
                jiraErrorTracker.getErrorCount(), exception)
        } catch (exception: Exception) {
            log.error("FromJiraUpdate stopped due to an error: jiraErrorCount={}, error={}",
                jiraErrorTracker.getErrorCount(), exception.message, exception)
        } finally {
            log.info(
                "Finished updated Task processing: received={}, existing={}, skipped={}, archived={}, affectedAgents={}, jiraErrorCount={}",
                receivedTasks, existingTasks, skippedTasks, archivedInitiatives,
                affectedAgentIds.size, jiraErrorTracker.getErrorCount()
            )
        }
    }

    /**
     * Сопоставляет Task страницы с jira_issue и обрабатывает их по инициативам.
     *
     * Повторную или неоднозначную связь по Jira key пропускает. Группировка
     * по agentId позволяет один раз проверить статус самой инициативы
     * и одной транзакцией сохранить изменения Task этой инициативы.
     */
    private fun processUpdatedTaskPage(
        tasks: List<SearchIssueDto>,
        initiativeStates: MutableMap<Long, JiraUpdatedInitiativeState>,
        processedTaskKeys: MutableSet<String>,
        jiraErrorTracker: JiraErrorTracker,
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
                        if (taskPersistenceService.saveUpdatedTasks(agentId, changes)) affectedAgentIds += agentId
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
     * «Отменена» и ClassicML приводят к архивированию. Если Jira GET
     * окончательно не удался либо статус отсутствует, Task не изменяются:
     * технический статус синхронизации становится error.
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
