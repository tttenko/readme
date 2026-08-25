```java

/**
 * Формирует запросы Jira Search для импорта инициатив.
 *
 * Содержит наборы Jira fields и правила формирования JQL
 * для поиска новых инициатив и Task monitoring epic.
 */
@Component
class JiraInitiativeSearchRequestFactory {

    companion object {

        private val INITIATIVE_FIELDS = listOf(
            "summary",
            "description",
            "status",
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

        private val MONITORING_TASK_FIELDS = listOf(
            "summary",
            "description",
            "status",
            "customfield_16700",
            "customfield_16701",
            "assignee",
            "reporter",
            "lastViewed",
            "resolutiondate",
            "created",
            "updated",
        )
    }

    /**
     * Формирует Jira Search запрос для поиска
     * новых инициатив.
     *
     * @param newDepth глубина поиска новых инициатив в днях
     * @param maxResults максимальное количество результатов на странице
     * @param startAt индекс первого элемента страницы
     * @return запрос Jira Search
     */
    fun createNewInitiativesRequest(
        newDepth: Int,
        maxResults: Int,
        startAt: Int,
    ): SearchIssueRequestDto {
        return SearchIssueRequestDto(
            fields = INITIATIVE_FIELDS,
            jql = buildNewInitiativesJql(newDepth),
            maxResults = maxResults,
            startAt = startAt,
        )
    }

    /**
     * Формирует Jira Search запрос для получения Task,
     * связанных с monitoring epic.
     *
     * @param epicKey Jira key monitoring epic
     * @param maxResults максимальное количество Task на странице
     * @param startAt индекс первого элемента страницы
     * @return запрос Jira Search
     */
    fun createMonitoringTasksRequest(
        epicKey: String,
        maxResults: Int,
        startAt: Int,
    ): SearchIssueRequestDto {
        return SearchIssueRequestDto(
            fields = MONITORING_TASK_FIELDS,
            jql = """
                project = CROSSGOAL
                AND "Epic Link" = $epicKey
            """
                .trimIndent()
                .replace("\n", " "),
            maxResults = maxResults,
            startAt = startAt,
        )
    }

    private fun buildNewInitiativesJql(
        newDepth: Int,
    ): String {
        return """
            project = CROSSGOAL
            AND issuetype = Инициатива
            AND resolution = Unresolved
            AND labels IN (AI_Native_портфель, AI-эффективность)
            AND created >= -${newDepth}d
            ORDER BY created DESC
        """
            .trimIndent()
            .replace("\n", " ")
    }
}




/**
 * Содержит идентификаторы monitoring epic,
 * найденного среди Jira issue links инициативы.
 */
data class JiraMonitoringEpicData(
    val jiraId: String?,
    val jiraKey: String,
)

/**
 * Определяет необходимость monitoring-синхронизации
 * и находит monitoring epic среди Jira issue links.
 */
@Component
class JiraMonitoringEpicResolver(
    private val jiraIssueKeyExtractor: JiraIssueKeyExtractor,
) {

    companion object {
        private const val AI_EFFECTIVENESS_LABEL = "AI-эффективность"
        private const val MONITORING_SUMMARY_PART = "мониторинг"

        private val log by logger()
    }

    /**
     * Определяет, требуется ли monitoring-синхронизация.
     *
     * Для инициатив с меткой AI-эффективность
     * monitoring epic и связанные Task не обрабатываются.
     *
     * @param labels Jira labels инициативы
     * @return true, если monitoring должен обрабатываться
     */
    fun isMonitoringRequired(
        labels: List<String>?,
    ): Boolean {
        return labels
            .orEmpty()
            .none { label ->
                label.equals(
                    other = AI_EFFECTIVENESS_LABEL,
                    ignoreCase = true,
                )
            }
    }

    /**
     * Ищет monitoring epic среди Jira issue links.
     *
     * Проверяются оба направления связи.
     * Monitoring-связью считается linked issue,
     * summary которого содержит слово "мониторинг"
     * без учёта регистра.
     *
     * @param initiativeJiraKey Jira key инициативы
     * @param issueLinks связи инициативы
     * @return monitoring epic либо null
     */
    fun findMonitoringEpic(
        initiativeJiraKey: String,
        issueLinks: List<GetIssueLinkResponse>?,
    ): JiraMonitoringEpicData? {
        val linkedIssue = issueLinks
            .orEmpty()
            .asSequence()
            .flatMap { issueLink ->
                sequenceOf(
                    issueLink.outwardIssue,
                    issueLink.inwardIssue,
                )
            }
            .filterNotNull()
            .firstOrNull { linkedIssue ->
                linkedIssue.fields
                    ?.summary
                    ?.contains(
                        other = MONITORING_SUMMARY_PART,
                        ignoreCase = true,
                    ) == true
            }
            ?: return null

        val epicKey =
            jiraIssueKeyExtractor.extractCrossgoalKey(linkedIssue.key)

        if (epicKey == null) {
            log.warn(
                "Monitoring epic has invalid CROSSGOAL key: initiativeKey={}, epicKey={}",
                initiativeJiraKey,
                linkedIssue.key,
            )

            return null
        }

        return JiraMonitoringEpicData(
            jiraId = linkedIssue.id,
            jiraKey = epicKey,
        )
    }
}




/**
 * Выполняет постраничный поиск всех Task,
 * связанных с monitoring epic.
 *
 * Использует общий JiraSearchPaginator,
 * поэтому логика пагинации не дублируется.
 *
 * Jira HTTP-вызовы выполняются без открытой
 * транзакции базы данных.
 */
@Component
class JiraMonitoringTaskSearchService(
    private val jiraSearchPaginator: JiraSearchPaginator,
    private val searchRequestFactory: JiraInitiativeSearchRequestFactory,
) {

    private val log by logger()

    /**
     * Получает все Task monitoring epic.
     *
     * Для расчёта текущего статуса инициативы необходим
     * полный набор Task, поэтому страницы объединяются
     * только в рамках одного epic.
     *
     * @param epicKey Jira key monitoring epic
     * @param maxResults размер одной Jira Search страницы
     * @return все найденные Task
     */
    fun searchMonitoringTasks(
        epicKey: String,
        maxResults: Int,
    ): List<SearchIssueDto> {
        val monitoringTasks = mutableListOf<SearchIssueDto>()

        jiraSearchPaginator.processPages(
            maxResults = maxResults,
            requestFactory = { startAt ->
                searchRequestFactory.createMonitoringTasksRequest(
                    epicKey = epicKey,
                    maxResults = maxResults,
                    startAt = startAt,
                )
            },
            pageProcessor = { response ->
                log.debug(
                    "Received monitoring tasks page: epicKey={}, issues={}, total={}",
                    epicKey,
                    response.issues.size,
                    response.total,
                )

                monitoringTasks.addAll(response.issues)
            },
        )

        log.info(
            "Finished loading monitoring tasks from Jira: epicKey={}, tasks={}",
            epicKey,
            monitoringTasks.size,
        )

        return monitoringTasks
    }
}





/**
 * Содержит Jira Task и соответствующий ему quality gate.
 *
 * Сопоставление выполняется один раз и далее
 * переиспользуется для обновления quality gates,
 * SLA, jira_issue и вычисления статуса инициативы.
 */
data class JiraTaskQualityGateMatch(
    val task: SearchIssueDto,
    val qualityGate: QualityGateEntity,
)

/**
 * Сопоставляет Jira Task со справочником quality_gate.
 *
 * Сопоставление выполняется по summary Task
 * и labels, заданным в quality_gate.regexp.
 */
@Component
class JiraTaskQualityGateMatcher {

    companion object {
        private val WHITESPACE_REGEX = Regex("\\s+")

        private val log by logger()
    }

    /**
     * Сопоставляет список Jira Task с quality gates.
     *
     * Task без найденного quality gate не включаются
     * в результирующий список.
     *
     * @param initiativeJiraKey Jira key инициативы
     * @param tasks Jira Task monitoring epic
     * @param qualityGates справочник quality_gate
     * @return найденные связи Task -> QualityGate
     */
    fun matchTasks(
        initiativeJiraKey: String,
        tasks: List<SearchIssueDto>,
        qualityGates: List<QualityGateEntity>,
    ): List<JiraTaskQualityGateMatch> {
        return tasks.mapNotNull { task ->
            val matchingQualityGate = findQualityGate(
                summary = task.fields?.summary,
                qualityGates = qualityGates,
            )

            if (matchingQualityGate == null) {
                log.debug(
                    "Quality gate was not found for monitoring Task: " +
                        "initiativeKey={}, taskKey={}, summary={}",
                    initiativeJiraKey,
                    task.key,
                    task.fields?.summary,
                )

                return@mapNotNull null
            }

            JiraTaskQualityGateMatch(
                task = task,
                qualityGate = matchingQualityGate,
            )
        }
    }

    /**
     * Находит quality gate по summary Jira Task.
     *
     * Основное правило:
     * все labels из regexp должны присутствовать в summary.
     *
     * Fallback по имени quality gate сохранён для совместимости
     * с существующей Jira synchronization logic.
     */
    private fun findQualityGate(
        summary: String?,
        qualityGates: List<QualityGateEntity>,
    ): QualityGateEntity? {
        val normalizedSummary =
            normalizeText(summary) ?: return null

        return qualityGates.firstOrNull { qualityGate ->
            val labels = qualityGate.regexp
                ?.split(",")
                ?.mapNotNull(::normalizeText)
                .orEmpty()

            labels.isNotEmpty() &&
                labels.all { label ->
                    normalizedSummary.contains(label)
                }
        } ?: qualityGates.firstOrNull { qualityGate ->
            normalizeText(qualityGate.name)
                ?.let { qualityGateName ->
                    normalizedSummary.contains(qualityGateName)
                }
                ?: false
        }
    }

    private fun normalizeText(
        value: String?,
    ): String? {
        return value
            ?.lowercase()
            ?.replace(WHITESPACE_REGEX, " ")
            ?.trim()
            ?.takeIf(String::isNotBlank)
    }
}



/**
 * Преобразует даты из Jira в LocalDateTime.
 */
@Component
class JiraDateTimeParser {

    companion object {
        private val JIRA_DATE_TIME_FORMATTER =
            DateTimeFormatter.ofPattern(
                "yyyy-MM-dd'T'HH:mm:ss.SSSZ"
            )
    }

    /**
     * Парсит дату Jira.
     *
     * Поддерживает:
     * - yyyy-MM-dd'T'HH:mm:ss.SSSZ;
     * - ISO_OFFSET_DATE_TIME;
     * - ISO_LOCAL_DATE.
     *
     * @param value Jira date value
     * @return LocalDateTime либо null
     */
    fun parse(
        value: String?,
    ): LocalDateTime? {
        if (value.isNullOrBlank()) {
            return null
        }

        return runCatching {
            OffsetDateTime
                .parse(
                    value,
                    JIRA_DATE_TIME_FORMATTER,
                )
                .toLocalDateTime()
        }.getOrNull()
            ?: runCatching {
                OffsetDateTime
                    .parse(
                        value,
                        DateTimeFormatter.ISO_OFFSET_DATE_TIME,
                    )
                    .toLocalDateTime()
            }.getOrNull()
            ?: runCatching {
                LocalDate
                    .parse(
                        value,
                        DateTimeFormatter.ISO_LOCAL_DATE,
                    )
                    .atStartOfDay()
            }.getOrNull()
    }
}


/**
 * Вычисляет текущий статус инициативы
 * по Jira Task, соответствующим этапам инициативы.
 */
@Component
class JiraInitiativeStatusResolver {

    companion object {
        private const val IN_PROGRESS_JIRA_STATUS_ID = "3"
        private const val TARGET_SOLUTION_STATUS_CODE = "targetSolution"

        private val TODO_JIRA_STATUS_IDS = setOf(
            "10109",
            "4",
            "10501",
        )
    }

    /**
     * Определяет текущий status code инициативы.
     *
     * Правила:
     * 1. если есть этапы InProgress — выбирается этап
     *    с максимальным ordering;
     * 2. иначе среди To Do/Reopened/Need Info выбирается
     *    этап с минимальным ordering;
     * 3. иначе возвращается targetSolution.
     *
     * @param taskMatches связи Jira Task с quality gates
     * @return рассчитанный status code
     */
    fun resolveStatusCode(
        taskMatches: List<JiraTaskQualityGateMatch>,
    ): String {
        val stageMatches = taskMatches.filter { match ->
            match.qualityGate.type == QualityGateType.status
        }

        val inProgressStatusCode = stageMatches
            .asSequence()
            .filter { match ->
                match.task.fields?.status?.id ==
                    IN_PROGRESS_JIRA_STATUS_ID
            }
            .maxByOrNull { match ->
                match.qualityGate.ordering ?: Int.MIN_VALUE
            }
            ?.qualityGate
            ?.status
            ?.code

        if (inProgressStatusCode != null) {
            return inProgressStatusCode
        }

        return stageMatches
            .asSequence()
            .filter { match ->
                match.task.fields?.status?.id in
                    TODO_JIRA_STATUS_IDS
            }
            .minByOrNull { match ->
                match.qualityGate.ordering ?: Int.MAX_VALUE
            }
            ?.qualityGate
            ?.status
            ?.code
            ?: TARGET_SOLUTION_STATUS_CODE
    }
}


AIAgentQualityGateRepository

Добавь в существующий repository этот метод:

/**
 * Создаёт отсутствующие или обновляет существующие
 * agent_quality_gate указанной инициативы.
 *
 * Используется при применении состояний,
 * рассчитанных по Jira Task.
 *
 * @param agentId идентификатор инициативы
 * @param qualityGateCodes коды обновляемых quality gates
 * @param state новое состояние quality gate
 * @param updatedAt дата изменения
 * @return количество изменённых записей
 */
@Transactional
@Modifying
@Query(
    value = """
        insert into agent_quality_gate (
            ai_agent_id,
            quality_gate_code,
            state,
            created,
            updated
        )
        select
            :agentId,
            quality_gate.code,
            :state,
            :updatedAt,
            :updatedAt
        from quality_gate
        where quality_gate.code in (:qualityGateCodes)
        on conflict (ai_agent_id, quality_gate_code)
        do update set
            state = excluded.state,
            updated = excluded.updated
    """,
    nativeQuery = true,
)
fun upsertStateForAgent(
    @Param("agentId")
    agentId: Long,

    @Param("qualityGateCodes")
    qualityGateCodes: Collection<String>,

    @Param("state")
    state: String,

    @Param("updatedAt")
    updatedAt: LocalDateTime,
): Int



/**
 * Сохраняет результат monitoring-синхронизации
 * новой Jira-инициативы.
 *
 * HTTP-вызовы Jira данным сервисом не выполняются.
 * Все изменения одной инициативы сохраняются
 * в коротких изолированных транзакциях.
 *
 * Monitoring epic сохраняется отдельной транзакцией
 * до выполнения Jira Search связанных Task.
 */
@Service
class JiraMonitoringPersistenceService(
    private val agentRepository: AIAgentRepository,
    private val jiraIssueRepository: JiraIssueRepository,
    private val agentStatusSlaRepository: AgentStatusSlaRepository,
    private val agentQualityGateRepository: AIAgentQualityGateRepository,
    private val jiraService: JiraService,
    private val jiraDateTimeParser: JiraDateTimeParser,
    private val initiativeStatusResolver: JiraInitiativeStatusResolver,
) {

    companion object {
        private const val CROSSGOAL_PROJECT = "crossgoal"

        private const val JIRA_FROM_STATUS_DONE = "done"
        private const val JIRA_FROM_STATUS_ERROR = "error"

        private val COMPLETED_TASK_STATUS_IDS = setOf(
            "10110",
            "5",
            "14103",
        )

        private val log by logger()
    }

    /**
     * Сохраняет связь инициативы с monitoring epic.
     *
     * Метод вызывается до Jira Search связанных Task.
     * Благодаря отдельной транзакции найденный epic
     * сохраняется независимо от результата последующей
     * загрузки monitoring Task.
     *
     * При повторном вызове существующая epic-связь
     * обновляется вместо создания дубликата.
     *
     * @param agentId id инициативы Пульта
     * @param monitoringEpic найденный monitoring epic
     * @return id jira_issue сохранённого monitoring epic
     */
    @Transactional(
        propagation = Propagation.REQUIRES_NEW,
        rollbackFor = [Exception::class],
    )
    fun saveMonitoringEpic(
        agentId: Long,
        monitoringEpic: JiraMonitoringEpicData,
    ): Long {
        val agent = agentRepository.getReferenceById(agentId)
        val currentDateTime = LocalDateTime.now()

        val epic = jiraIssueRepository
            .findByAgentIdAndTypeAndProject(
                agentId,
                JiraIssueType.epic.name,
                CROSSGOAL_PROJECT,
            )
            .firstOrNull()
            ?: JiraIssueEntity(
                agent = agent,
                project = CROSSGOAL_PROJECT,
                type = JiraIssueType.epic.name,
            ).apply {
                created = currentDateTime
            }

        epic.jiraId = monitoringEpic.jiraId
        epic.jiraKey = monitoringEpic.jiraKey
        epic.jiraUrl =
            jiraService.getJiraSigmaUrl() +
                monitoringEpic.jiraKey

        val savedEpic = jiraIssueRepository.save(epic)

        log.debug(
            "Saved monitoring epic before Task synchronization: " +
                "agentId={}, epicIssueId={}, epicKey={}",
            agentId,
            savedEpic.id,
            monitoringEpic.jiraKey,
        )

        return savedEpic.id
    }

    /**
     * Сохраняет результат обработки monitoring Task.
     *
     * Monitoring epic к этому моменту уже сохранён
     * отдельной транзакцией.
     *
     * В одной транзакции:
     * - обновляются agent_quality_gate;
     * - создаются/обновляются agent_status_sla;
     * - создаются jira_issue типа task;
     * - вычисляется agent_status;
     * - Jira synchronization переводится в done.
     *
     * @param agentId id инициативы Пульта
     * @param initiativeJiraKey Jira key инициативы
     * @param monitoringEpicIssueId id сохранённого jira_issue monitoring epic
     * @param monitoringEpicKey Jira key monitoring epic
     * @param taskMatches сопоставленные Task и quality gates
     * @param referenceData справочники текущего запуска
     */
    @Transactional(
        propagation = Propagation.REQUIRES_NEW,
        rollbackFor = [Exception::class],
    )
    fun saveMonitoringData(
        agentId: Long,
        initiativeJiraKey: String,
        monitoringEpicIssueId: Long,
        monitoringEpicKey: String,
        taskMatches: List<JiraTaskQualityGateMatch>,
        referenceData: JiraImportReferenceData,
    ) {
        val agent = agentRepository.findByIdForUpdate(agentId)
            ?: error("AI agent with id=$agentId was not found")

        val currentDateTime = LocalDateTime.now()

        updateQualityGates(
            agentId = agent.id,
            taskMatches = taskMatches,
            currentDateTime = currentDateTime,
        )

        saveStatusSla(
            agent = agent,
            taskMatches = taskMatches,
        )

        saveTaskIssues(
            agent = agent,
            monitoringEpicIssueId = monitoringEpicIssueId,
            monitoringEpicKey = monitoringEpicKey,
            taskMatches = taskMatches,
            currentDateTime = currentDateTime,
        )

        val currentStatusCode =
            initiativeStatusResolver.resolveStatusCode(
                taskMatches = taskMatches,
            )

        val currentStatus =
            referenceData.statusesByCode[currentStatusCode]

        if (currentStatus == null) {
            log.error(
                "Calculated initiative status was not found in reference data: " +
                    "jiraKey={}, agentId={}, statusCode={}",
                initiativeJiraKey,
                agent.id,
                currentStatusCode,
            )

            error(
                "Status '$currentStatusCode' was not found in Jira import reference data"
            )
        }

        agent.agentStatus = currentStatus
        agent.jiraFromStatus = JIRA_FROM_STATUS_DONE
        agent.jiraUpdated = currentDateTime
        agent.updated = currentDateTime

        agentRepository.save(agent)

        log.info(
            "Monitoring synchronization completed successfully: " +
                "jiraKey={}, agentId={}, epicKey={}, matchedTasks={}, currentStatus={}",
            initiativeJiraKey,
            agent.id,
            monitoringEpicKey,
            taskMatches.size,
            currentStatusCode,
        )
    }

    /**
     * Завершает Jira synchronization без monitoring.
     *
     * Используется для AI-эффективности и инициатив,
     * у которых monitoring epic отсутствует.
     *
     * Текущий agent_status не изменяется.
     *
     * @param agentId id инициативы
     * @param jiraKey Jira key инициативы
     */
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    fun markSynchronizationDone(
        agentId: Long,
        jiraKey: String,
    ) {
        updateSynchronizationStatus(
            agentId = agentId,
            jiraKey = jiraKey,
            jiraFromStatus = JIRA_FROM_STATUS_DONE,
        )
    }

    /**
     * Помечает Jira synchronization ошибкой.
     *
     * Текущий agent_status не изменяется,
     * поэтому новая инициатива остаётся в analysis.
     *
     * @param agentId id инициативы
     * @param jiraKey Jira key инициативы
     */
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    fun markSynchronizationError(
        agentId: Long,
        jiraKey: String,
    ) {
        updateSynchronizationStatus(
            agentId = agentId,
            jiraKey = jiraKey,
            jiraFromStatus = JIRA_FROM_STATUS_ERROR,
        )
    }

    /**
     * Обновляет состояния вех инициативы
     * на основании статусов Jira Task.
     */
    private fun updateQualityGates(
        agentId: Long,
        taskMatches: List<JiraTaskQualityGateMatch>,
        currentDateTime: LocalDateTime,
    ) {
        val qualityGateMatches = taskMatches.filter { match ->
            match.qualityGate.type ==
                QualityGateType.quality_gate
        }

        val checkedQualityGateCodes = qualityGateMatches
            .asSequence()
            .filter { match ->
                match.task.fields?.status?.id in
                    COMPLETED_TASK_STATUS_IDS
            }
            .mapNotNull { match ->
                match.qualityGate.code
            }
            .toSet()

        val uncheckedQualityGateCodes = qualityGateMatches
            .asSequence()
            .filter { match ->
                match.task.fields?.status?.id !in
                    COMPLETED_TASK_STATUS_IDS
            }
            .mapNotNull { match ->
                match.qualityGate.code
            }
            .filterNot(checkedQualityGateCodes::contains)
            .toSet()

        upsertQualityGateState(
            agentId = agentId,
            qualityGateCodes = checkedQualityGateCodes,
            state = QualityGateState.checked,
            currentDateTime = currentDateTime,
        )

        upsertQualityGateState(
            agentId = agentId,
            qualityGateCodes = uncheckedQualityGateCodes,
            state = QualityGateState.unchecked,
            currentDateTime = currentDateTime,
        )

        log.debug(
            "Updated quality gates from monitoring Task: " +
                "agentId={}, checked={}, unchecked={}",
            agentId,
            checkedQualityGateCodes.size,
            uncheckedQualityGateCodes.size,
        )
    }

    private fun upsertQualityGateState(
        agentId: Long,
        qualityGateCodes: Set<String>,
        state: QualityGateState,
        currentDateTime: LocalDateTime,
    ) {
        if (qualityGateCodes.isEmpty()) {
            return
        }

        agentQualityGateRepository.upsertStateForAgent(
            agentId = agentId,
            qualityGateCodes = qualityGateCodes,
            state = state.name,
            updatedAt = currentDateTime,
        )
    }

    /**
     * Сохраняет плановые и фактические даты
     * завершения этапов инициативы.
     *
     * Существующие SLA загружаются одним запросом.
     */
    private fun saveStatusSla(
        agent: AIAgentEntity,
        taskMatches: List<JiraTaskQualityGateMatch>,
    ) {
        val stageMatches = taskMatches.filter { match ->
            match.qualityGate.type == QualityGateType.status
        }

        if (stageMatches.isEmpty()) {
            log.debug(
                "No stage Task found for SLA synchronization: agentId={}",
                agent.id,
            )

            return
        }

        val existingSlaByStatusId =
            agentStatusSlaRepository
                .findAllByAiAgentId(agent.id)
                .associateBy { sla ->
                    sla.primaryKey.agentStatusId
                }
                .toMutableMap()

        val slaByStatusId =
            linkedMapOf<Long, AgentStatusSlaEntity>()

        stageMatches.forEach { match ->
            val status = match.qualityGate.status

            if (status?.id == null) {
                log.warn(
                    "Cannot synchronize SLA because quality gate has no status: " +
                        "agentId={}, taskKey={}, qualityGateCode={}",
                    agent.id,
                    match.task.key,
                    match.qualityGate.code,
                )

                return@forEach
            }

            val plannedDateValue =
                match.task.fields?.customfield_16701

            val completedDateValue =
                match.task.fields?.resolutiondate

            val plannedDate =
                jiraDateTimeParser.parse(plannedDateValue)

            val completedDate =
                jiraDateTimeParser.parse(completedDateValue)

            if (
                !plannedDateValue.isNullOrBlank() &&
                plannedDate == null
            ) {
                log.warn(
                    "Cannot parse Jira planned date: agentId={}, taskKey={}, value={}",
                    agent.id,
                    match.task.key,
                    plannedDateValue,
                )
            }

            if (
                !completedDateValue.isNullOrBlank() &&
                completedDate == null
            ) {
                log.warn(
                    "Cannot parse Jira completed date: agentId={}, taskKey={}, value={}",
                    agent.id,
                    match.task.key,
                    completedDateValue,
                )
            }

            val statusId = status.id

            val sla = existingSlaByStatusId[statusId]
                ?: AgentStatusSlaEntity().apply {
                    aiAgent = agent
                    agentStatus = status
                }

            sla.plannedDate = plannedDate
            sla.completedDate = completedDate

            slaByStatusId[statusId] = sla
        }

        if (slaByStatusId.isNotEmpty()) {
            agentStatusSlaRepository.saveAll(
                slaByStatusId.values
            )
        }

        log.debug(
            "Saved monitoring SLA: agentId={}, slaCount={}",
            agent.id,
            slaByStatusId.size,
        )
    }

    /**
     * Создаёт jira_issue для Task,
     * сопоставленных с quality gates.
     *
     * Существующие jira_issue загружаются одним batch-запросом.
     *
     * @param monitoringEpicIssueId id сохранённого monitoring epic
     * @param monitoringEpicKey Jira key monitoring epic для логирования
     */
    private fun saveTaskIssues(
        agent: AIAgentEntity,
        monitoringEpicIssueId: Long,
        monitoringEpicKey: String,
        taskMatches: List<JiraTaskQualityGateMatch>,
        currentDateTime: LocalDateTime,
    ) {
        val jiraKeys = taskMatches
            .mapNotNull { match ->
                match.task.key
            }
            .toSet()

        if (jiraKeys.isEmpty()) {
            log.debug(
                "No matched Jira Task to persist: agentId={}, epicKey={}",
                agent.id,
                monitoringEpicKey,
            )

            return
        }

        val existingIssuesByJiraKey =
            jiraIssueRepository
                .findAllByAgentIdAndJiraKeyIn(
                    agent.id,
                    jiraKeys,
                )
                .mapNotNull { jiraIssue ->
                    jiraIssue.jiraKey
                        ?.let { jiraKey ->
                            jiraKey to jiraIssue
                        }
                }
                .toMap()

        val taskIssues = taskMatches.mapNotNull { match ->
            val jiraKey = match.task.key
                ?.takeIf(String::isNotBlank)
                ?: return@mapNotNull null

            existingIssuesByJiraKey[jiraKey]
                ?.apply {
                    jiraId = match.task.id
                    jiraUrl =
                        jiraService.getJiraSigmaUrl() +
                            jiraKey
                    parentId = monitoringEpicIssueId
                    qualityGate = match.qualityGate
                }
                ?: JiraIssueEntity(
                    agent = agent,
                    project = CROSSGOAL_PROJECT,
                    type = JiraIssueType.task.name,
                    jiraId = match.task.id,
                    jiraKey = jiraKey,
                    jiraUrl =
                        jiraService.getJiraSigmaUrl() +
                            jiraKey,
                    parentId = monitoringEpicIssueId,
                    qualityGate = match.qualityGate,
                ).apply {
                    created = currentDateTime
                }
        }

        if (taskIssues.isNotEmpty()) {
            jiraIssueRepository.saveAll(taskIssues)
        }

        log.debug(
            "Saved Jira monitoring Task relations: " +
                "agentId={}, epicKey={}, taskCount={}",
            agent.id,
            monitoringEpicKey,
            taskIssues.size,
        )
    }

    /**
     * Обновляет технический статус Jira synchronization.
     */
    private fun updateSynchronizationStatus(
        agentId: Long,
        jiraKey: String,
        jiraFromStatus: String,
    ) {
        val agent = agentRepository.findByIdForUpdate(agentId)
            ?: error("AI agent with id=$agentId was not found")

        val currentDateTime = LocalDateTime.now()

        agent.jiraFromStatus = jiraFromStatus
        agent.jiraUpdated = currentDateTime
        agent.updated = currentDateTime

        agentRepository.save(agent)

        log.debug(
            "Updated Jira synchronization status: " +
                "jiraKey={}, agentId={}, jiraFromStatus={}",
            jiraKey,
            agentId,
            jiraFromStatus,
        )
    }
}



/**
 * Оркестрирует синхронизацию monitoring epic
 * новой инициативы в рамках FR1.
 *
 * Последовательность:
 * - проверяет необходимость monitoring;
 * - находит monitoring epic;
 * - сохраняет monitoring epic в отдельной транзакции;
 * - получает все Jira Task;
 * - сопоставляет Task с quality gates;
 * - передаёт результат в транзакционный persistence service.
 *
 * Jira HTTP-вызовы выполняются вне DB-транзакции.
 */
@Service
class JiraNewInitiativeMonitoringService(
    private val monitoringEpicResolver: JiraMonitoringEpicResolver,
    private val monitoringTaskSearchService: JiraMonitoringTaskSearchService,
    private val taskQualityGateMatcher: JiraTaskQualityGateMatcher,
    private val monitoringPersistenceService: JiraMonitoringPersistenceService,
) {

    private val log by logger()

    /**
     * Выполняет monitoring-синхронизацию новой инициативы.
     *
     * Отсутствие monitoring epic и AI-эффективность
     * являются нормальными сценариями и завершаются
     * jiraFromStatus=done без изменения agent_status.
     *
     * Найденный monitoring epic сохраняется до Jira Search Task,
     * поэтому связь с epic не теряется при ошибке получения Task.
     *
     * Ошибка получения/обработки Task приводит
     * к jiraFromStatus=error.
     *
     * @param agentId id созданной инициативы
     * @param issue исходная Jira-инициатива
     * @param referenceData справочники текущего запуска
     * @param maxResults размер Jira Search страницы
     * @return true при успешной обработке, false при ошибке
     */
    fun synchronizeMonitoring(
        agentId: Long,
        issue: SearchIssueDto,
        referenceData: JiraImportReferenceData,
        maxResults: Int,
    ): Boolean {
        val jiraKey = requireNotNull(issue.key) {
            "Jira initiative key must not be null"
        }

        val monitoringRequired =
            monitoringEpicResolver.isMonitoringRequired(
                labels = issue.fields?.labels,
            )

        if (!monitoringRequired) {
            log.debug(
                "Monitoring is not required for AI-effectiveness initiative: " +
                    "jiraKey={}, agentId={}",
                jiraKey,
                agentId,
            )

            monitoringPersistenceService.markSynchronizationDone(
                agentId = agentId,
                jiraKey = jiraKey,
            )

            return true
        }

        val monitoringEpic =
            monitoringEpicResolver.findMonitoringEpic(
                initiativeJiraKey = jiraKey,
                issueLinks = issue.fields?.issuelinks,
            )

        if (monitoringEpic == null) {
            log.debug(
                "Monitoring epic was not found for Jira initiative: " +
                    "jiraKey={}, agentId={}",
                jiraKey,
                agentId,
            )

            monitoringPersistenceService.markSynchronizationDone(
                agentId = agentId,
                jiraKey = jiraKey,
            )

            return true
        }

        log.info(
            "Started monitoring synchronization: " +
                "jiraKey={}, agentId={}, epicKey={}",
            jiraKey,
            agentId,
            monitoringEpic.jiraKey,
        )

        return try {
            /*
             * Epic сохраняется до Jira Search Task.
             * Транзакция внутри saveMonitoringEpic завершается
             * до выполнения сетевого вызова.
             */
            val monitoringEpicIssueId =
                monitoringPersistenceService.saveMonitoringEpic(
                    agentId = agentId,
                    monitoringEpic = monitoringEpic,
                )

            val monitoringTasks =
                monitoringTaskSearchService.searchMonitoringTasks(
                    epicKey = monitoringEpic.jiraKey,
                    maxResults = maxResults,
                )

            val taskMatches =
                taskQualityGateMatcher.matchTasks(
                    initiativeJiraKey = jiraKey,
                    tasks = monitoringTasks,
                    qualityGates = referenceData.qualityGates,
                )

            log.info(
                "Monitoring Task matching completed: " +
                    "jiraKey={}, epicKey={}, receivedTasks={}, matchedTasks={}",
                jiraKey,
                monitoringEpic.jiraKey,
                monitoringTasks.size,
                taskMatches.size,
            )

            monitoringPersistenceService.saveMonitoringData(
                agentId = agentId,
                initiativeJiraKey = jiraKey,
                monitoringEpicIssueId = monitoringEpicIssueId,
                monitoringEpicKey = monitoringEpic.jiraKey,
                taskMatches = taskMatches,
                referenceData = referenceData,
            )

            true

        } catch (exception: Exception) {
            log.error(
                "Failed to synchronize monitoring data: " +
                    "jiraKey={}, agentId={}, epicKey={}, error={}",
                jiraKey,
                agentId,
                monitoringEpic.jiraKey,
                exception.message,
                exception,
            )

            monitoringPersistenceService.markSynchronizationError(
                agentId = agentId,
                jiraKey = jiraKey,
            )

            false
        }
    }
}



```
