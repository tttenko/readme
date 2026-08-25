```java

/**
 * Оркестрирует поиск и создание новых Jira-инициатив в рамках FR1.
 *
 * Получает настройки и справочники, выполняет постраничный Jira Search,
 * применяет ограничения FR1, проверяет существование инициатив,
 * создаёт новые инициативы и запускает обработку monitoring epic.
 *
 * Jira-инициативы обрабатываются независимо друг от друга,
 * поэтому ошибка одной инициативы не прерывает обработку остальных.
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
     * Выполняет поиск и создание новых инициатив из Jira.
     *
     * Справочники загружаются один раз на запуск.
     * Jira-результаты обрабатываются постранично без накопления
     * всех инициатив в памяти.
     *
     * Для каждой новой инициативы:
     * 1. создаются основные данные инициативы;
     * 2. после завершения транзакции создания запускается
     *    обработка monitoring epic и связанных Jira Task.
     */
    fun importNewInitiatives() {
        log.info("Started importing new initiatives from Jira")

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

        var receivedInitiatives = 0
        var existingInitiatives = 0
        var newInitiatives = 0
        var createdInitiatives = 0
        var ignoredInitiatives = 0
        var failedInitiatives = 0

        jiraSearchPaginator.processPages(
            maxResults = maxResults,
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
                    "Received Jira initiatives page: issues={}, total={}",
                    issues.size,
                    response.total,
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
                existingInitiatives += pageStatistics.existingInitiatives
                newInitiatives += pageStatistics.newInitiatives
                createdInitiatives += pageStatistics.createdInitiatives
                ignoredInitiatives += pageStatistics.ignoredInitiatives
                failedInitiatives += pageStatistics.failedInitiatives
            },
        )

        log.info(
            "Finished importing new initiatives from Jira: " +
                "received={}, existing={}, new={}, created={}, ignored={}, failed={}",
            receivedInitiatives,
            existingInitiatives,
            newInitiatives,
            createdInitiatives,
            ignoredInitiatives,
            failedInitiatives,
        )
    }

    /**
     * Обрабатывает одну страницу Jira Search.
     *
     * Последовательность обработки:
     * 1. исключает инициативы с некорректным CROSSGOAL key;
     * 2. исключает инициативы согласно ограничениям FR1;
     * 3. одним batch-запросом проверяет существование оставшихся инициатив;
     * 4. создаёт отсутствующие инициативы в отдельных транзакциях;
     * 5. для успешно созданных инициатив запускает monitoring-синхронизацию.
     *
     * Новые инициативы обрабатываются независимо друг от друга,
     * поэтому ошибка одной инициативы не прерывает обработку страницы.
     *
     * @param issues инициативы текущей страницы Jira Search
     * @param referenceData справочники текущего запуска scheduler
     * @param maxResults размер страницы Jira Search для последующей
     * загрузки monitoring Task
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

        val initiativesByJiraKey = buildMap<String, SearchIssueDto> {
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

                put(
                    normalizedJiraKey,
                    issue,
                )
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
                    jiraKeys = initiativesByJiraKey.keys,
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
                normalizedJiraKey,
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
                        normalizedJiraKey,
                    )

                    return@forEach
                }

                createdInitiatives++

                log.debug(
                    "Jira initiative created, starting monitoring synchronization: " +
                        "jiraKey={}, agentId={}",
                    normalizedJiraKey,
                    createdAgentId,
                )

                val monitoringSynchronized =
                    jiraNewInitiativeMonitoringService.synchronizeMonitoring(
                        agentId = createdAgentId,
                        issue = issue,
                        referenceData = referenceData,
                        maxResults = maxResults,
                    )

                if (!monitoringSynchronized) {
                    failedInitiatives++

                    log.warn(
                        "Jira initiative was created but monitoring synchronization failed: " +
                            "jiraKey={}, agentId={}",
                        normalizedJiraKey,
                        createdAgentId,
                    )
                }
            } catch (exception: Exception) {
                failedInitiatives++

                log.error(
                    "Failed to process new Jira initiative: jiraKey={}, error={}",
                    normalizedJiraKey,
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
     * - если её Jira-статус "Отменена";
     * - если среди Jira labels присутствует ClassicML.
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

        val hasClassicMlLabel = issue.fields
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
 * Используется для формирования общей статистики
 * выполнения scheduler.
 *
 * @property existingInitiatives количество уже существующих инициатив
 * @property newInitiatives количество найденных новых инициатив
 * @property createdInitiatives количество успешно созданных инициатив
 * @property ignoredInitiatives количество пропущенных инициатив
 * @property failedInitiatives количество инициатив с ошибкой обработки
 */
data class JiraInitiativePageStatistics(
    val existingInitiatives: Int = 0,
    val newInitiatives: Int = 0,
    val createdInitiatives: Int = 0,
    val ignoredInitiatives: Int = 0,
    val failedInitiatives: Int = 0,
)
```
