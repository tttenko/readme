```java

/** Формирует Jira Search запрос для изменённых задач мониторинга. */
@Component
class JiraUpdatedTaskSearchRequestFactory {

    companion object {
        private val TASK_FIELDS = listOf(
            "summary", "description", "status", "customfield_16700", "customfield_16701",
            "assignee", "reporter", "lastViewed", "resolutiondate", "created", "updated"
        )
    }

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
}

/** Ищет уже связанные с инициативами Пульта задачи мониторинга. */
@Repository
interface JiraUpdatedTaskRepository : Repository<JiraIssueEntity, Long> {

    @Query(
        """
        select jiraIssue
        from JiraIssueEntity jiraIssue
        where jiraIssue.jiraKey in :jiraKeys
          and jiraIssue.type = 'task'
          and lower(jiraIssue.project) = 'crossgoal'
          and jiraIssue.agent is not null
        """
    )
    fun findExistingMonitoringTasks(@Param("jiraKeys") jiraKeys: Collection<String>): List<JiraIssueEntity>
}

/** Оркестрирует поиск и обработку обновлений Jira в рамках FR2. */
@Service
class JiraInitiativeUpdateService(
    private val optionsService: OptionsService,
    private val searchRequestFactory: JiraUpdatedTaskSearchRequestFactory,
    private val jiraSearchPaginator: JiraSearchPaginator,
    private val jiraIssueKeyExtractor: JiraIssueKeyExtractor,
    private val updatedTaskRepository: JiraUpdatedTaskRepository,
) {

    companion object {
        private val log by logger()
    }

    // Счётчик принадлежит только FR2 и сбрасывается на каждый запуск.
    private val jiraErrorTracker = JiraErrorTracker()

    fun synchronizeUpdates() {
        jiraErrorTracker.reset()

        var receivedTasks = 0
        var existingTasks = 0
        var skippedTasks = 0

        log.info("Started FromJiraUpdate scheduler: jiraErrorCount={}", jiraErrorTracker.getErrorCount())

        try {
            val options = optionsService.getCurrent()
            val updateDepth = requireNotNull(options.updateDepth) { "Jira updateDepth is not configured" }
            val maxResults = requireNotNull(options.maxResults) { "Jira maxResults is not configured" }

            require(updateDepth > 0) { "Jira updateDepth must be positive" }
            require(maxResults > 0) { "Jira maxResults must be positive" }

            log.info("Started searching updated monitoring Tasks: updateDepth={}, maxResults={}", updateDepth, maxResults)

            jiraSearchPaginator.processPages(
                maxResults = maxResults,
                jiraErrorTracker = jiraErrorTracker,
                requestFactory = { startAt ->
                    searchRequestFactory.createUpdatedTasksRequest(updateDepth, maxResults, startAt)
                },
                pageProcessor = { response ->
                    if (response.total == 0) {
                        log.info("No updated monitoring Tasks found in Jira")
                    }

                    val pageStatistics = processUpdatedTaskPage(response.issues)
                    receivedTasks += response.issues.size
                    existingTasks += pageStatistics.existingTasks
                    skippedTasks += pageStatistics.skippedTasks
                }
            )

            // Этап 3: здесь будет запущен второй подпроцесс — поиск обновлённых инициатив.
        } catch (exception: JiraErrorLimitExceededException) {
            log.error(
                "FromJiraUpdate scheduler stopped because Jira error limit was exceeded: jiraErrorCount={}",
                jiraErrorTracker.getErrorCount(), exception
            )
        } catch (exception: Exception) {
            log.error(
                "FromJiraUpdate scheduler stopped due to an error: jiraErrorCount={}, error={}",
                jiraErrorTracker.getErrorCount(), exception.message, exception
            )
        } finally {
            log.info(
                "Finished searching updated monitoring Tasks: received={}, existing={}, skipped={}, jiraErrorCount={}",
                receivedTasks, existingTasks, skippedTasks, jiraErrorTracker.getErrorCount()
            )
            jiraErrorTracker.reset()
        }
    }

    /** Проверяет Task текущей страницы одним запросом к jira_issue. */
    private fun processUpdatedTaskPage(tasks: List<SearchIssueDto>): JiraUpdatedTaskPageStatistics {
        if (tasks.isEmpty()) return JiraUpdatedTaskPageStatistics()

        val jiraKeys = tasks.mapNotNull { task -> jiraIssueKeyExtractor.extractCrossgoalKey(task.key) }.toSet()
        val savedTasksByJiraKey = if (jiraKeys.isEmpty()) {
            emptyMap()
        } else {
            updatedTaskRepository.findExistingMonitoringTasks(jiraKeys).groupBy { savedTask -> savedTask.jiraKey }
        }

        var existingTasks = 0
        var skippedTasks = 0

        tasks.forEach { task ->
            val jiraKey = jiraIssueKeyExtractor.extractCrossgoalKey(task.key)
            val savedTasks = savedTasksByJiraKey[jiraKey].orEmpty()

            if (jiraKey == null || savedTasks.isEmpty()) {
                skippedTasks++
                log.debug("Skipping updated Jira Task absent from Pult: jiraKey={}", task.key)
                return@forEach
            }

            if (savedTasks.size > 1) {
                skippedTasks++
                log.warn("Skipping updated Jira Task with ambiguous Pult relations: jiraKey={}, relations={}", jiraKey, savedTasks.size)
                return@forEach
            }

            existingTasks++
            log.debug("Found updated Jira Task in Pult: jiraKey={}, jiraIssueId={}", jiraKey, savedTasks.single().id)

            // Этап 2: передать task и savedTasks.single() в обработчик QG/SLA.
        }

        return JiraUpdatedTaskPageStatistics(existingTasks = existingTasks, skippedTasks = skippedTasks)
    }
}

/** Статистика проверки одной страницы изменённых Task. */
data class JiraUpdatedTaskPageStatistics(
    val existingTasks: Int = 0,
    val skippedTasks: Int = 0,
)

/** Запускает FR2 по расписанию с распределённой блокировкой. */
@Component
class JiraInitiativeUpdateScheduler(
    private val updateService: JiraInitiativeUpdateService,
) {

    companion object {
        const val LOCK_NAME = "FromJiraUpdateScheduler"
        private const val ZONE = "Europe/Moscow"
    }

    @Scheduled(cron = "\${scheduled.jira-sync.from-jira-update-cron}", zone = ZONE)
    @SchedulerLock(
        name = LOCK_NAME,
        lockAtLeastFor = "\${scheduled.jira-sync.lock-at-least-for}",
        lockAtMostFor = "\${scheduled.jira-sync.lock-at-most-for}",
    )
    fun runScheduledUpdate() {
        updateService.synchronizeUpdates()
    }
}

/** Запускает FR2 вручную с той же блокировкой, что и запуск по расписанию. */
@Component
class JiraInitiativeUpdateManual(
    private val updateService: JiraInitiativeUpdateService,
    private val schedulerProperties: JiraSchedulerProperties,
    private val lockingTaskExecutor: LockingTaskExecutor,
) {

    private val log by logger()

    @Async
    fun run() {
        val lockAtLeastFor = schedulerProperties.lockAtLeastFor ?: Duration.ofSeconds(10)
        val lockAtMostFor = schedulerProperties.lockAtMostFor ?: Duration.ofSeconds(15)

        lockingTaskExecutor.executeWithLock(
            Runnable {
                log.info("Started manual FromJiraUpdate scheduler execution")
                updateService.synchronizeUpdates()
            },
            LockConfiguration(
                Instant.now(),
                JiraInitiativeUpdateScheduler.LOCK_NAME,
                lockAtMostFor,
                lockAtLeastFor,
            )
        )
    }
}

/** Ручной запуск шедулера обновления инициатив из Jira. */
@RestController
@RequestMapping("/api/v1/admin/scheduler")
class AdminFromJiraUpdateSchedulerController(
    private val fromJiraUpdateManual: JiraInitiativeUpdateManual,
) {

    @PostMapping("/from-jira-update/run")
    @PreAuthorize("hasAnyAuthority('CMS_ADMIN')")
    @PlatformAudit(
        dataType = "scheduler-from-jira-update",
        actionType = Run,
        value = "Ручной запуск шедулера обновления инициатив из Jira"
    )
    fun runFromJiraUpdateScheduler(): ResponseEntity<Void> {
        fromJiraUpdateManual.run()
        return ResponseEntity.accepted().build()
    }
}




```
