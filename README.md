```java

/**
 * Формирует запросы Jira Search для синхронизации инициатив.
 *
 * Содержит набор запрашиваемых Jira-полей и правила построения JQL,
 * необходимые для поиска инициатив.
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
    }

    /**
     * Формирует запрос для поиска новых инициатив в Jira.
     *
     * @param newDepth глубина поиска новых инициатив в днях
     * @param maxResults максимальное количество элементов на одной странице
     * @param startAt индекс первого элемента запрашиваемой страницы
     * @return запрос для Jira Search
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
     * Формирует JQL для поиска новых инициатив CROSSGOAL
     * за указанное количество последних дней.
     *
     * @param newDepth глубина поиска в днях
     * @return готовый JQL-запрос
     */
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
 * Выполняет постраничный поиск данных через Jira Search.
 *
 * Последовательно запрашивает страницы и сразу передаёт каждую
 * полученную страницу вызывающему коду для обработки.
 */
@Component
class JiraSearchPaginator(
    private val jiraService: JiraService,
) {

    /**
     * Получает и обрабатывает все доступные страницы Jira Search.
     *
     * Не накапливает результаты всех страниц в памяти.
     *
     * @param maxResults размер одной страницы
     * @param requestFactory функция формирования запроса для текущего startAt
     * @param pageProcessor функция обработки полученной страницы
     */
    fun processPages(
        maxResults: Int,
        requestFactory: (startAt: Int) -> SearchIssueRequestDto,
        pageProcessor: (SearchIssueResponseDto) -> Unit,
    ) {
        var startAt = 0

        while (true) {
            val response = jiraService.search(
                requestFactory(startAt)
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
}

/**
 * Нормализует имена энейблеров перед их сопоставлением.
 *
 * Используется для одинаковой обработки значений из Jira
 * и справочника Пульта.
 */
@Component
class EnablerNameNormalizer {

    companion object {
        private val WHITESPACE_REGEX = Regex("\\s+")
    }

    /**
     * Приводит имя к нижнему регистру и удаляет все пробельные символы.
     *
     * Пустое или отсутствующее значение преобразуется в null.
     *
     * @param name исходное имя энейблера
     * @return нормализованное имя либо null
     */
    fun normalize(
        name: String?,
    ): String? {
        return name
            ?.lowercase()
            ?.replace(WHITESPACE_REGEX, "")
            ?.takeIf(String::isNotBlank)
    }
}

EnablerRepository
/**
 * Возвращает энейблеры, доступные для использования в синхронизации.
 *
 * Записи с disabled=true исключаются.
 */
@Query(
    """
        select enabler
        from EnablerEntity enabler
        where enabler.disabled = false
           or enabler.disabled is null
    """
)
fun findAllActive(): List<EnablerEntity>
StatusRepository
/**
 * Возвращает активные статусы инициатив.
 *
 * Записи с disabled=true исключаются.
 */
@Query(
    """
        select status
        from StatusEntity status
        where status.disabled = false
           or status.disabled is null
    """
)
fun findAllActive(): List<StatusEntity>
QualityGateRepository
/**
 * Возвращает активные Quality Gates для обработки Jira-инициатив.
 *
 * Записи с disabled=true исключаются.
 */
@Query(
    """
        select qualityGate
        from QualityGateEntity qualityGate
        where qualityGate.disabled = false
           or qualityGate.disabled is null
    """
)
fun findAllActive(): List<QualityGateEntity>

/**
 * Содержит справочные данные, используемые в рамках одного запуска Jira-импорта.
 *
 * Данные предварительно подготавливаются для быстрого сопоставления
 * при последующей обработке инициатив.
 */
data class JiraImportReferenceData(
    val strategies: List<StrategyEntity>,
    val enablersByNormalizedName: Map<String, EnablerEntity>,
    val statusesByCode: Map<String, StatusEntity>,
    val qualityGates: List<QualityGateEntity>,
)

/**
 * Загружает и подготавливает справочные данные для Jira-импорта.
 *
 * Справочники читаются один раз на запуск синхронизации и затем
 * переиспользуются при обработке всех полученных инициатив.
 */
@Component
class JiraImportReferenceDataProvider(
    private val strategyService: StrategyService,
    private val enablerRepository: EnablerRepository,
    private val statusRepository: StatusRepository,
    private val qualityGateRepository: QualityGateRepository,
    private val enablerNameNormalizer: EnablerNameNormalizer,
) {

    private val log by logger()

    /**
     * Загружает необходимые справочники и подготавливает их
     * для дальнейшего поиска по ключевым значениям.
     *
     * @return подготовленные справочные данные текущего запуска
     */
    fun load(): JiraImportReferenceData {
        val strategies = strategyService.findAll()

        val enablersByNormalizedName = enablerRepository
            .findAllActive()
            .mapNotNull { enabler ->
                enablerNameNormalizer
                    .normalize(enabler.name)
                    ?.let { normalizedName ->
                        normalizedName to enabler
                    }
            }
            .toMap()

        val statusesByCode = statusRepository
            .findAllActive()
            .mapNotNull { status ->
                status.code?.let { statusCode ->
                    statusCode to status
                }
            }
            .toMap()

        val qualityGates = qualityGateRepository.findAllActive()

        log.debug(
            "Loaded Jira import reference data: strategies={}, enablers={}, statuses={}, qualityGates={}",
            strategies.size,
            enablersByNormalizedName.size,
            statusesByCode.size,
            qualityGates.size,
        )

        return JiraImportReferenceData(
            strategies = strategies,
            enablersByNormalizedName = enablersByNormalizedName,
            statusesByCode = statusesByCode,
            qualityGates = qualityGates,
        )
    }
}

interface OptionsRepository : JpaRepository<OptionsEntity, Long> {

    /**
     * Возвращает первую запись с настройками scheduler.
     *
     * Таблица options используется как источник текущей конфигурации
     * глубины поиска и размера страницы Jira.
     */
    fun findFirstByOrderByIdAsc(): OptionsEntity?
}

/**
 * Предоставляет операции чтения и изменения настроек scheduler.
 */
@Service
class OptionsService(
    val repository: OptionsRepository,
    private val validationTemplate: ValidationTemplate,
) {

    /**
     * Возвращает все сохранённые настройки.
     *
     * @return список настроек
     */
    fun findAll(): List<OptionsDto> {
        return repository.findAll()
            .map { it.toOptionsDto() }
    }

    /**
     * Возвращает текущие настройки, используемые scheduler.
     *
     * Если конфигурация отсутствует, выполнение завершается ошибкой.
     *
     * @return текущие настройки scheduler
     */
    fun getCurrent(): OptionsDto {
        return repository.findFirstByOrderByIdAsc()
            ?.toOptionsDto()
            ?: error("Jira synchronization options are not configured")
    }

    /**
     * Обновляет переданные настройки после их валидации.
     *
     * Не переданные значения сохраняются без изменений.
     *
     * @param dto новые значения настроек
     */
    fun update(
        dto: OptionsDto,
    ) {
        validationTemplate.assertValid(dto)

        val entity = repository.findFirstByOrderByIdAsc()
            ?: OptionsEntity()

        entity.apply {
            dto.newDepth?.let { newDepth = it }
            dto.updateDepth?.let { updateDepth = it }
            dto.maxResults?.let { maxResults = it }
        }

        repository.save(entity)
    }
}

/**
 * Выполняет read-only поиск существующих Jira-инициатив.
 *
 * Проверяет наличие CROSSGOAL-ключей одновременно в jira_issue
 * и в Jira-ссылке основной сущности ai_agent.
 */
@Repository
interface ExistingJiraInitiativeRepository : Repository<JiraIssueEntity, Long> {

    /**
     * Возвращает Jira-ключи, которые уже представлены в Пульте.
     *
     * Проверка выполняется одним batch-запросом для всей переданной коллекции,
     * без загрузки AIAgentEntity и JiraIssueEntity в persistence context.
     *
     * @param jiraKeys CROSSGOAL-ключи найденных в Jira инициатив
     * @return ключи уже существующих инициатив
     */
    @Query(
        value = """
            select existing_key
            from (
                select jira_issue.jira_key as existing_key
                from jira_issue
                where jira_issue.jira_key in (:jiraKeys)

                union

                select substring(
                    upper(ai_agent.agent_jira_url)
                    from '(CROSSGOAL-[0-9]+)'
                ) as existing_key
                from ai_agent
                where ai_agent.agent_jira_url is not null
                  and substring(
                        upper(ai_agent.agent_jira_url)
                        from '(CROSSGOAL-[0-9]+)'
                      ) in (:jiraKeys)
            ) existing_initiatives
        """,
        nativeQuery = true,
    )
    fun findExistingJiraKeys(
        @Param("jiraKeys")
        jiraKeys: Collection<String>,
    ): List<String>
}

/**
 * Содержит статистику обработки одной страницы Jira Search.
 *
 * Используется для формирования общей статистики выполнения scheduler.
 */
data class JiraInitiativePageStatistics(
    val existingInitiatives: Int = 0,
    val newInitiatives: Int = 0,
    val ignoredInitiatives: Int = 0,
)

/**
 * Оркестрирует поиск новых инициатив в Jira в рамках FR1.
 *
 * Получает настройки и справочники, запускает постраничный Jira Search,
 * фильтрует неподходящие инициативы и определяет уже существующие в Пульте.
 */
@Service
class JiraNewInitiativeSearchSchedulerService(
    private val optionsService: OptionsService,
    private val referenceDataProvider: JiraImportReferenceDataProvider,
    private val searchRequestFactory: JiraInitiativeSearchRequestFactory,
    private val jiraSearchPaginator: JiraSearchPaginator,
    private val existingJiraInitiativeRepository: ExistingJiraInitiativeRepository,
) {

    companion object {
        private const val CANCELLED_STATUS_NAME = "Отменена"
        private const val CLASSIC_ML_LABEL = "ClassicML"
        private const val CROSSGOAL_KEY_PREFIX = "CROSSGOAL-"

        private val log by logger()
    }

    /**
     * Выполняет поиск и предварительный отбор новых Jira-инициатив.
     *
     * Обрабатывает результаты постранично и формирует статистику запуска.
     * На текущем этапе новые инициативы только определяются, но ещё не сохраняются.
     */
    fun importNewInitiatives() {
        log.info("Started searching for new initiatives in Jira")

        val options = optionsService.getCurrent()

        val newDepth = requireNotNull(options.newDepth) {
            "New initiatives search depth is not configured"
        }

        val maxResults = requireNotNull(options.maxResults) {
            "Jira search maxResults is not configured"
        }

        /*
         * Данные понадобятся при создании инициатив на следующем этапе FR1.
         * Загружаем их один раз на весь запуск.
         */
        val referenceData = referenceDataProvider.load()

        log.debug(
            "Jira reference data prepared: strategies={}, enablers={}, statuses={}, qualityGates={}",
            referenceData.strategies.size,
            referenceData.enablersByNormalizedName.size,
            referenceData.statusesByCode.size,
            referenceData.qualityGates.size,
        )

        var receivedInitiatives = 0
        var existingInitiatives = 0
        var newInitiatives = 0
        var ignoredInitiatives = 0

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

                val pageStatistics = processPage(issues)

                receivedInitiatives += issues.size
                existingInitiatives += pageStatistics.existingInitiatives
                newInitiatives += pageStatistics.newInitiatives
                ignoredInitiatives += pageStatistics.ignoredInitiatives
            },
        )

        log.info(
            "Finished searching for new initiatives in Jira: received={}, existing={}, new={}, ignored={}",
            receivedInitiatives,
            existingInitiatives,
            newInitiatives,
            ignoredInitiatives,
        )
    }

    /**
     * Обрабатывает одну страницу Jira-инициатив.
     *
     * Проверяет ключи и ограничения FR1, после чего одним запросом
     * определяет инициативы, которые уже существуют в Пульте.
     *
     * @param issues инициативы текущей страницы Jira Search
     * @return статистика обработки страницы
     */
    private fun processPage(
        issues: List<SearchIssueDto>,
    ): JiraInitiativePageStatistics {
        if (issues.isEmpty()) {
            return JiraInitiativePageStatistics()
        }

        var ignoredInitiatives = 0

        val initiativesByJiraKey = buildMap<String, SearchIssueDto> {
            issues.forEach { issue ->
                val jiraKey = normalizeJiraKey(issue.key)

                if (jiraKey == null) {
                    ignoredInitiatives++

                    log.warn(
                        "Skipping Jira initiative because CROSSGOAL key is missing or invalid: key={}",
                        issue.key,
                    )

                    return@forEach
                }

                if (shouldSkipInitiative(issue)) {
                    ignoredInitiatives++
                    return@forEach
                }

                put(jiraKey, issue)
            }
        }

        if (initiativesByJiraKey.isEmpty()) {
            return JiraInitiativePageStatistics(
                ignoredInitiatives = ignoredInitiatives,
            )
        }

        val existingJiraKeys = existingJiraInitiativeRepository
            .findExistingJiraKeys(
                jiraKeys = initiativesByJiraKey.keys,
            )
            .toSet()

        var existingInitiatives = 0
        var newInitiatives = 0

        initiativesByJiraKey.forEach { (jiraKey, issue) ->
            if (jiraKey in existingJiraKeys) {
                existingInitiatives++

                log.debug(
                    "Initiative with Jira key {} already exists in Pult",
                    jiraKey,
                )

                return@forEach
            }

            newInitiatives++

            log.debug(
                "New Jira initiative found: {}",
                jiraKey,
            )

            /*
             * Следующий этап FR1:
             *
             * jiraNewInitiativeCreator.create(
             *     jiraIssue = issue,
             *     referenceData = referenceData,
             * )
             */
        }

        return JiraInitiativePageStatistics(
            existingInitiatives = existingInitiatives,
            newInitiatives = newInitiatives,
            ignoredInitiatives = ignoredInitiatives,
        )
    }

    /**
     * Проверяет ограничения, при которых новая инициатива
     * не должна добавляться в Пульт.
     *
     * Сейчас исключаются отменённые инициативы и инициативы
     * с меткой ClassicML.
     *
     * @param issue Jira-инициатива
     * @return true, если инициативу необходимо пропустить
     */
    private fun shouldSkipInitiative(
        issue: SearchIssueDto,
    ): Boolean {
        val jiraKey = issue.key

        val jiraStatusName = issue.fields
            ?.status
            ?.name

        if (
            jiraStatusName.equals(
                CANCELLED_STATUS_NAME,
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
                    CLASSIC_ML_LABEL,
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

    /**
     * Нормализует Jira-ключ и проверяет принадлежность проекту CROSSGOAL.
     *
     * Ключ приводится к верхнему регистру и очищается от внешних пробелов.
     *
     * @param jiraKey исходный ключ Jira
     * @return нормализованный CROSSGOAL-ключ либо null
     */
    private fun normalizeJiraKey(
        jiraKey: String?,
    ): String? {
        return jiraKey
            ?.trim()
            ?.uppercase()
            ?.takeIf { normalizedJiraKey ->
                normalizedJiraKey.startsWith(CROSSGOAL_KEY_PREFIX)
            }
    }
}

/**
 * Запускает поиск новых Jira-инициатив по расписанию.
 *
 * ShedLock гарантирует, что FR1 одновременно выполняется
 * только в одном экземпляре приложения.
 */
@Component
class JiraNewInitiativeSearchScheduler(
    private val fromJiraNewService: FromJiraNewService,
) {

    companion object {
        const val LOCK_NAME = "FromJiraNewScheduler"

        private const val ZONE = "Europe/Moscow"
    }

    /**
     * Запускает FR1 согласно настроенному cron-расписанию.
     */
    @Scheduled(
        cron = "\${scheduled.jira-sync.from-jira-new-cron}",
        zone = ZONE,
    )
    @SchedulerLock(
        name = LOCK_NAME,
        lockAtLeastFor = "\${scheduled.jira-sync.lock-at-least-for}",
        lockAtMostFor = "\${scheduled.jira-sync.lock-at-most-for}",
    )
    fun runScheduledImport() {
        fromJiraNewService.importNewInitiatives()
    }
}

/**
 * Выполняет ручной запуск поиска новых Jira-инициатив.
 *
 * Использует тот же ShedLock, что и scheduled-запуск,
 * поэтому два варианта запуска не могут выполняться одновременно.
 */
@Component
class FromJiraNewManual(
    private val fromJiraNewService: FromJiraNewService,
    private val schedulerProperties: JiraSchedulerProperties,
    private val lockingTaskExecutor: LockingTaskExecutor,
) {

    private val log by logger()

    /**
     * Асинхронно запускает FR1 с распределённой блокировкой.
     *
     * Параметры времени блокировки берутся из общей конфигурации scheduler.
     */
    @Async
    fun run() {
        val lockAtLeastFor = requireNotNull(
            schedulerProperties.lockAtLeastFor
        ) {
            "Jira scheduler lockAtLeastFor is not configured"
        }

        val lockAtMostFor = requireNotNull(
            schedulerProperties.lockAtMostFor
        ) {
            "Jira scheduler lockAtMostFor is not configured"
        }

        lockingTaskExecutor.executeWithLock(
            Runnable {
                log.info("Started manual FromJiraNew scheduler execution")

                fromJiraNewService.importNewInitiatives()
            },
            LockConfiguration(
                Instant.now(),
                FromJiraNewScheduler.LOCK_NAME,
                lockAtMostFor,
                lockAtLeastFor,
            ),
        )
    }
}

/**
 * Предоставляет административный API для ручного запуска FR1.
 */
@RestController
@RequestMapping("/api/ai/v1/admin/scheduler")
class FromJiraNewSchedulerController(
    private val fromJiraNewManual: FromJiraNewManual,
) {

    /**
     * Запускает поиск новых инициатив в Jira вручную.
     *
     * Выполнение начинается асинхронно, поэтому endpoint
     * сразу возвращает HTTP 202 Accepted.
     */
    @PostMapping("/from-jira-new/run")
    fun runFromJiraNewScheduler(): ResponseEntity<Void> {
        fromJiraNewManual.run()

        return ResponseEntity.accepted().build()
    }
}


```
