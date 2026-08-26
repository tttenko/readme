```java

/**
 * Сигнализирует о превышении допустимого количества
 * ошибок Jira в рамках одного запуска scheduler.
 *
 * Исключение используется для немедленной остановки текущего
 * сценария синхронизации и не является ошибкой отдельной инициативы.
 */
class JiraErrorLimitExceededException(
    cause: Throwable? = null,
) : RuntimeException(
    "Jira error limit exceeded",
    cause,
)

/**
 * Выполняет постраничный Jira Search.
 *
 * Последовательно запрашивает страницы и сразу передаёт каждую
 * полученную страницу вызывающему коду для обработки.
 *
 * Повторные попытки Jira-запроса выполняются существующим
 * механизмом @Retryable на InternalJiraFeignClient.
 *
 * Если Jira-запрос завершился ошибкой после исчерпания retry,
 * увеличивается общий jiraErrorCount текущего запуска scheduler.
 */
@Component
class JiraSearchPaginator(
    private val jiraService: JiraService,
) {

    private val log by logger()

    /**
     * Получает и обрабатывает все доступные страницы Jira Search.
     *
     * Не накапливает результаты всех страниц в памяти.
     *
     * @param maxResults размер одной страницы
     * @param jiraErrorTracker счётчик ошибок Jira текущего запуска scheduler
     * @param requestFactory функция формирования запроса для текущего startAt
     * @param pageProcessor функция обработки успешно полученной страницы
     */
    fun processPages(
        maxResults: Int,
        jiraErrorTracker: JiraErrorTracker,
        requestFactory: (Int) -> SearchIssueRequestDto,
        pageProcessor: (SearchIssueResponseDto) -> Unit,
    ) {
        var startAt = 0

        while (true) {
            val request = requestFactory(startAt)

            val response = try {
                jiraService.search(request)
            } catch (exception: Exception) {
                handleJiraSearchError(
                    startAt = startAt,
                    jiraErrorTracker = jiraErrorTracker,
                    exception = exception,
                )
            }

            log.info(
                "Jira Search page received successfully: startAt={}, issues={}, total={}",
                startAt,
                response.issues.size,
                response.total,
            )

            pageProcessor(response)

            val total = response.total ?: return
            val issues = response.issues

            if (
                issues.isEmpty() ||
                startAt + maxResults >= total
            ) {
                return
            }

            startAt += maxResults
        }
    }

    /**
     * Обрабатывает окончательную ошибку Jira Search
     * после выполнения всех настроенных retry.
     *
     * Ошибка увеличивает общий jiraErrorCount текущего запуска.
     * При превышении лимита выбрасывается специальное исключение,
     * которое должно остановить scheduler.
     */
    private fun handleJiraSearchError(
        startAt: Int,
        jiraErrorTracker: JiraErrorTracker,
        exception: Exception,
    ): Nothing {
        val jiraErrorCount = jiraErrorTracker.increment()

        log.error(
            "Jira Search failed after all retries: " +
                "startAt={}, jiraErrorCount={}, maxJiraErrors={}, error={}",
            startAt,
            jiraErrorCount,
            JiraErrorTracker.MAX_JIRA_ERRORS,
            exception.message,
            exception,
        )

        if (jiraErrorTracker.isErrorLimitExceeded()) {
            log.error(
                "Jira error limit exceeded: jiraErrorCount={}, maxJiraErrors={}",
                jiraErrorCount,
                JiraErrorTracker.MAX_JIRA_ERRORS,
            )

            throw JiraErrorLimitExceededException(exception)
        }

        throw exception
    }
}

/**
 * Выполняет постраничный поиск всех Task,
 * связанных с monitoring epic.
 *
 * Использует общий JiraSearchPaginator,
 * поэтому логика пагинации и обработки Jira-ошибок
 * не дублируется.
 *
 * Jira HTTP-вызовы выполняются без открытой DB-транзакции.
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
     * @param jiraErrorTracker счётчик Jira-ошибок текущего запуска scheduler
     * @return все найденные Task
     */
    fun searchMonitoringTasks(
        epicKey: String,
        maxResults: Int,
        jiraErrorTracker: JiraErrorTracker,
    ): List<SearchIssueDto> {
        val monitoringTasks = mutableListOf<SearchIssueDto>()

        log.info(
            "Started loading monitoring tasks from Jira: epicKey={}",
            epicKey,
        )

        jiraSearchPaginator.processPages(
            maxResults = maxResults,
            jiraErrorTracker = jiraErrorTracker,
            requestFactory = { startAt ->
                searchRequestFactory.createMonitoringTasksRequest(
                    epicKey = epicKey,
                    maxResults = maxResults,
                    startAt = startAt,
                )
            },
            pageProcessor = { response ->
                log.debug(
                    "Received monitoring tasks page: " +
                        "epicKey={}, issues={}, total={}",
                    epicKey,
                    response.issues.size,
                    response.total,
                )

                monitoringTasks.addAll(response.issues)
            },
        )

        log.info(
            "Finished loading monitoring tasks from Jira: " +
                "epicKey={}, tasks={}",
            epicKey,
            monitoringTasks.size,
        )

        return monitoringTasks
    }
}

/**
 * Оркестрирует синхронизацию monitoring epic
 * новой инициативы в рамках FR1.
 *
 * Последовательность:
 * - проверяет необходимость monitoring;
 * - находит monitoring epic;
 * - сохраняет найденную связь с epic;
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
     * Ошибка получения или обработки Task приводит
     * к jiraFromStatus=error.
     *
     * Если общий лимит Jira-ошибок превышен,
     * текущая инициатива помечается error,
     * после чего исключение передаётся наверх для остановки scheduler.
     *
     * @param agentId id созданной инициативы
     * @param issue исходная Jira-инициатива
     * @param referenceData справочники текущего запуска
     * @param maxResults размер Jira Search страницы
     * @param jiraErrorTracker счётчик Jira-ошибок текущего запуска
     * @return true при успешной обработке, false при ошибке конкретной инициативы
     */
    fun synchronizeMonitoring(
        agentId: Long,
        issue: SearchIssueDto,
        referenceData: JiraImportReferenceData,
        maxResults: Int,
        jiraErrorTracker: JiraErrorTracker,
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

        val monitoringEpicIssueId =
            monitoringPersistenceService.saveMonitoringEpic(
                agentId = agentId,
                monitoringEpic = monitoringEpic,
            )

        return try {
            val monitoringTasks =
                monitoringTaskSearchService.searchMonitoringTasks(
                    epicKey = monitoringEpic.jiraKey,
                    maxResults = maxResults,
                    jiraErrorTracker = jiraErrorTracker,
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
                monitoringEpicIssueId = monitoringEpicIssueId,
                monitoringEpicKey = monitoringEpic.jiraKey,
                initiativeJiraKey = jiraKey,
                taskMatches = taskMatches,
                referenceData = referenceData,
            )

            true
        } catch (exception: JiraErrorLimitExceededException) {
            log.error(
                "Jira error limit exceeded during monitoring synchronization: " +
                    "jiraKey={}, agentId={}, epicKey={}, jiraErrorCount={}",
                jiraKey,
                agentId,
                monitoringEpic.jiraKey,
                jiraErrorTracker.getErrorCount(),
                exception,
            )

            markSynchronizationErrorSafely(
                agentId = agentId,
                jiraKey = jiraKey,
            )

            throw exception
        } catch (exception: Exception) {
            log.error(
                "Failed to synchronize monitoring data: " +
                    "jiraKey={}, agentId={}, epicKey={}, jiraErrorCount={}, error={}",
                jiraKey,
                agentId,
                monitoringEpic.jiraKey,
                jiraErrorTracker.getErrorCount(),
                exception.message,
                exception,
            )

            markSynchronizationErrorSafely(
                agentId = agentId,
                jiraKey = jiraKey,
            )

            false
        }
    }

    /**
     * Сохраняет error-state инициативы после ошибки monitoring.
     *
     * Ошибка обновления технического статуса логируется отдельно,
     * чтобы не скрыть первоначальную ошибку Jira или обработки данных.
     */
    private fun markSynchronizationErrorSafely(
        agentId: Long,
        jiraKey: String,
    ) {
        try {
            monitoringPersistenceService.markSynchronizationError(
                agentId = agentId,
                jiraKey = jiraKey,
            )
        } catch (exception: Exception) {
            log.error(
                "Failed to persist Jira synchronization error state: " +
                    "jiraKey={}, agentId={}, error={}",
                jiraKey,
                agentId,
                exception.message,
                exception,
            )
        }
    }
}

/**
 * Оркестрирует поиск и создание новых Jira-инициатив в рамках FR1.
 *
 * Получает настройки и справочники, выполняет постраничный Jira Search,
 * применяет ограничения FR1, проверяет существование инициатив,
 * создаёт новые инициативы и выполняет monitoring-синхронизацию.
 *
 * jiraErrorCount относится только к одному запуску FR1
 * и сбрасывается перед началом и после завершения scheduler.
 */
@Service
class JiraNewInitiativeImportService(
    private val optionsService: OptionsService,
    private val jiraIssueKeyExtractor: JiraIssueKeyExtractor,
    private val referenceDataProvider: JiraImportReferenceDataProvider,
    private val searchRequestFactory: JiraInitiativeSearchRequestFactory,
    private val jiraSearchPaginator: JiraSearchPaginator,
    private val existingJiraInitiativeRepository: ExistingJiraInitiativeRepository,
    private val jiraNewInitiativeCreator: JiraNewInitiativeCreationService,
    private val jiraNewInitiativeMonitoringService: JiraNewInitiativeMonitoringService,
) {

    companion object {
        private const val CANCELLED_STATUS_NAME = "Отменена"
        private const val CLASSIC_ML_LABEL = "ClassicML"

        private val log by logger()
    }

    /**
     * Отдельный счётчик ошибок FR1.
     *
     * Не используется как Spring dependency намеренно:
     * существующий независимый Jira scheduler имеет собственный
     * жизненный цикл и не должен влиять на jiraErrorCount FR1.
     *
     * Одновременные scheduled/manual запуски FR1 запрещены
     * единым ShedLock.
     */
    private val jiraErrorTracker = JiraErrorTracker()

    /**
     * Выполняет поиск и создание новых инициатив из Jira.
     *
     * Справочники загружаются один раз на запуск.
     * Jira-результаты обрабатываются постранично без накопления
     * всех инициатив в памяти.
     *
     * При окончательной ошибке основного Jira Search scheduler
     * завершает работу.
     *
     * При превышении jiraErrorCount > 15 scheduler
     * немедленно прекращает дальнейшую обработку.
     */
    fun importNewInitiatives() {
        jiraErrorTracker.reset()

        log.info(
            "Started importing new initiatives from Jira: jiraErrorCount={}",
            jiraErrorTracker.getErrorCount(),
        )

        var receivedInitiatives = 0
        var existingInitiatives = 0
        var newInitiatives = 0
        var createdInitiatives = 0
        var ignoredInitiatives = 0
        var failedInitiatives = 0

        try {
            val options = optionsService.getCurrent()

            val newDepth = requireNotNull(options.newDepth) {
                "Jira new initiatives search depth is not configured"
            }

            val maxResults = requireNotNull(options.maxResults) {
                "Jira maxResults is not configured"
            }

            val referenceData = referenceDataProvider.load()

            log.debug(
                "Jira reference data prepared: " +
                    "strategies={}, enablers={}, statuses={}, qualityGates={}, " +
                    "divisions={}, blocks={}, initiativeTypes={}",
                referenceData.strategiesByJiraKey.size,
                referenceData.enablersByNormalizedName.size,
                referenceData.statusesByCode.size,
                referenceData.qualityGates.size,
                referenceData.divisionsByLabel.size,
                referenceData.blocksByLabel.size,
                referenceData.initiativeTypesByCode.size,
            )

            jiraSearchPaginator.processPages(
                maxResults = maxResults,
                jiraErrorTracker = jiraErrorTracker,
                requestFactory = { startAt ->
                    searchRequestFactory.createNewInitiativesRequest(
                        newDepth = newDepth,
                        maxResults = maxResults,
                        startAt = startAt,
                    )
                },
                pageProcessor = { response ->
                    val issues = response.issues

                    log.info(
                        "Received Jira initiatives page: " +
                            "issues={}, total={}, jiraErrorCount={}",
                        issues.size,
                        response.total,
                        jiraErrorTracker.getErrorCount(),
                    )

                    if (response.total == 0) {
                        log.info("No new initiatives found in Jira")
                        return@processPages
                    }

                    val pageStatistics = processPage(
                        issues = issues,
                        referenceData = referenceData,
                        maxResults = maxResults,
                    )

                    receivedInitiatives += issues.size
                    existingInitiatives +=
                        pageStatistics.existingInitiatives
                    newInitiatives +=
                        pageStatistics.newInitiatives
                    createdInitiatives +=
                        pageStatistics.createdInitiatives
                    ignoredInitiatives +=
                        pageStatistics.ignoredInitiatives
                    failedInitiatives +=
                        pageStatistics.failedInitiatives
                },
            )
        } catch (exception: JiraErrorLimitExceededException) {
            log.error(
                "FromJiraNew scheduler stopped because Jira error limit was exceeded: " +
                    "jiraErrorCount={}, maxJiraErrors={}",
                jiraErrorTracker.getErrorCount(),
                JiraErrorTracker.MAX_JIRA_ERRORS,
                exception,
            )
        } catch (exception: Exception) {
            log.error(
                "FromJiraNew scheduler was stopped due to an error: " +
                    "jiraErrorCount={}, error={}",
                jiraErrorTracker.getErrorCount(),
                exception.message,
                exception,
            )
        } finally {
            log.info(
                "Finished importing new initiatives from Jira: " +
                    "received={}, existing={}, new={}, created={}, ignored={}, failed={}, " +
                    "jiraErrorCount={}",
                receivedInitiatives,
                existingInitiatives,
                newInitiatives,
                createdInitiatives,
                ignoredInitiatives,
                failedInitiatives,
                jiraErrorTracker.getErrorCount(),
            )

            jiraErrorTracker.reset()
        }
    }

    /**
     * Обрабатывает одну страницу Jira Search.
     *
     * Сначала исключает неподходящие инициативы,
     * затем одним batch-запросом проверяет существование оставшихся ключей.
     *
     * Каждая новая инициатива создаётся в отдельной транзакции,
     * после чего отдельно выполняется monitoring-синхронизация.
     *
     * @param issues инициативы текущей Jira Search страницы
     * @param referenceData справочники текущего запуска
     * @param maxResults размер Jira Search страницы
     * @return статистика обработки текущей страницы
     */
    private fun processPage(
        issues: List<SearchIssueDto>,
        referenceData: JiraImportReferenceData,
        maxResults: Int,
    ): JiraInitiativePageStatistics {
        if (issues.isEmpty()) {
            return JiraInitiativePageStatistics()
        }

        var ignoredInitiatives = 0

        val initiativesByJiraKey =
            buildMap<String, SearchIssueDto> {
                issues.forEach { issue ->
                    val normalizedJiraKey =
                        jiraIssueKeyExtractor.extractCrossgoalKey(
                            issue.key
                        )

                    if (normalizedJiraKey == null) {
                        ignoredInitiatives++

                        log.warn(
                            "Skipping Jira initiative because CROSSGOAL key " +
                                "is missing or invalid: key={}",
                            issue.key,
                        )

                        return@forEach
                    }

                    if (shouldSkipInitiative(issue)) {
                        ignoredInitiatives++
                        return@forEach
                    }

                    put(normalizedJiraKey, issue)
                }
            }

        if (initiativesByJiraKey.isEmpty()) {
            return JiraInitiativePageStatistics(
                ignoredInitiatives = ignoredInitiatives,
            )
        }

        val existingJiraKeys =
            existingJiraInitiativeRepository
                .findExistingJiraKeys(
                    jiraKeys = initiativesByJiraKey.keys
                )
                .toSet()

        var existingInitiatives = 0
        var newInitiatives = 0
        var createdInitiatives = 0
        var failedInitiatives = 0

        initiativesByJiraKey.forEach { (normalizedJiraKey, issue) ->
            if (normalizedJiraKey in existingJiraKeys) {
                existingInitiatives++

                log.debug(
                    "Initiative with Jira key {} already exists in Pult",
                    normalizedJiraKey,
                )

                return@forEach
            }

            newInitiatives++

            log.debug(
                "New Jira initiative found, starting creation: jiraKey={}",
                issue.key,
            )

            try {
                val createdAgentId =
                    jiraNewInitiativeCreator.createInitiativeFromJira(
                        issue = issue,
                        referenceData = referenceData,
                    )

                if (createdAgentId == null) {
                    ignoredInitiatives++

                    log.debug(
                        "Jira initiative was skipped during creation: jiraKey={}",
                        issue.key,
                    )

                    return@forEach
                }

                createdInitiatives++

                val monitoringSynchronized =
                    jiraNewInitiativeMonitoringService
                        .synchronizeMonitoring(
                            agentId = createdAgentId,
                            issue = issue,
                            referenceData = referenceData,
                            maxResults = maxResults,
                            jiraErrorTracker = jiraErrorTracker,
                        )

                if (!monitoringSynchronized) {
                    failedInitiatives++
                }
            } catch (exception: JiraErrorLimitExceededException) {
                /*
                 * Глобальный лимит Jira-ошибок нельзя обрабатывать
                 * как ошибку одной инициативы.
                 *
                 * Исключение должно подняться до importNewInitiatives()
                 * и остановить весь FR1.
                 */
                throw exception
            } catch (exception: Exception) {
                failedInitiatives++

                log.error(
                    "Failed to process new Jira initiative: " +
                        "jiraKey={}, error={}",
                    issue.key,
                    exception.message,
                    exception,
                )
            }
        }

        return JiraInitiativePageStatistics(
            existingInitiatives = existingInitiatives,
            newInitiatives = newInitiatives,
            createdInitiatives = createdInitiatives,
            ignoredInitiatives = ignoredInitiatives,
            failedInitiatives = failedInitiatives,
        )
    }

    /**
     * Проверяет ограничения FR1, при которых новая инициатива
     * не должна быть добавлена в Пульт.
     *
     * Инициатива пропускается:
     * - если её Jira-статус равен "Отменена";
     * - если среди labels присутствует ClassicML.
     *
     * @param issue Jira-инициатива
     * @return true, если инициативу необходимо пропустить
     */
    private fun shouldSkipInitiative(
        issue: SearchIssueDto,
    ): Boolean {
        val jiraKey = issue.key
        val jiraStatusName = issue.fields?.status?.name

        if (
            jiraStatusName.equals(
                other = CANCELLED_STATUS_NAME,
                ignoreCase = true,
            )
        ) {
            log.debug(
                "Skipping Jira initiative {} because its status is {}",
                jiraKey,
                jiraStatusName,
            )

            return true
        }

        val hasClassicMlLabel =
            issue.fields
                ?.labels
                .orEmpty()
                .any { label ->
                    label.equals(
                        other = CLASSIC_ML_LABEL,
                        ignoreCase = true,
                    )
                }

        if (hasClassicMlLabel) {
            log.debug(
                "Skipping Jira initiative {} because it contains label {}",
                jiraKey,
                CLASSIC_ML_LABEL,
            )

            return true
        }

        return false
    }
}

/**
 * Содержит статистику обработки одной страницы Jira Search.
 *
 * Используется для формирования общей статистики выполнения scheduler.
 */
data class JiraInitiativePageStatistics(
    val existingInitiatives: Int = 0,
    val newInitiatives: Int = 0,
    val createdInitiatives: Int = 0,
    val ignoredInitiatives: Int = 0,
    val failedInitiatives: Int = 0,
)


```
