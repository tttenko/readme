```java
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="
            http://www.liquibase.org/xml/ns/dbchangelog
            http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.5.xsd">

    <changeSet id="add created into involved_resource" author="Maxim">

        <addColumn tableName="involved_resource">
            <column
                    name="created"
                    type="DATETIME"
                    defaultValueComputed="CURRENT_TIMESTAMP"
                    remarks="дата и время создания записи">
                <constraints nullable="false"/>
            </column>
        </addColumn>

        <rollback>
            <dropColumn
                    tableName="involved_resource"
                    columnName="created"/>
        </rollback>

    </changeSet>

</databaseChangeLog>

/**
     * Дата и время создания записи о задействованном ресурсе.
     *
     * Инициализируется при создании entity. Значение также имеет
     * DEFAULT CURRENT_TIMESTAMP на уровне БД для обратной совместимости
     * с существующими сценариями вставки.
     */
    @Column(name = "created", nullable = false, updatable = false)
    var created: LocalDateTime = LocalDateTime.now()

/**
 * Извлекает Jira issue key из строковых значений.
 *
 * Поддерживает как непосредственно Jira key,
 * так и строки или URL, содержащие Jira key.
 */
@Component
class JiraIssueKeyExtractor {

    companion object {
        private const val CROSSGOAL_KEY_PREFIX = "CROSSGOAL-"

        private val JIRA_KEY_REGEX =
            Regex("""[A-Z][A-Z0-9]+-\d+""")
    }

    /**
     * Извлекает Jira key из переданного значения
     * и приводит его к верхнему регистру.
     *
     * @param value исходная строка
     * @return найденный Jira key либо null
     */
    fun extractJiraKey(value: String?): String? {
        return value
            ?.uppercase()
            ?.let { normalizedValue ->
                JIRA_KEY_REGEX.find(normalizedValue)?.value
            }
    }

    /**
     * Извлекает Jira key проекта CROSSGOAL.
     *
     * @param value исходная строка
     * @return CROSSGOAL key либо null, если значение отсутствует,
     * содержит некорректный Jira key или относится к другому проекту
     */
    fun extractCrossgoalKey(value: String?): String? {
        return extractJiraKey(value)
            ?.takeIf { jiraKey ->
                jiraKey.startsWith(CROSSGOAL_KEY_PREFIX)
            }
    }
}

/**
 * Извлекает числовые значения из текстовых Jira-полей.
 *
 * Поддерживает целые и дробные значения с точкой или запятой
 * в качестве десятичного разделителя.
 */
@Component
class JiraNumericValueParser {

    companion object {
        private val NUMBER_REGEX = Regex("""-?\d+(?:[.,]\d+)?""")
    }

    /**
     * Возвращает первое найденное в строке числовое значение.
     *
     * @param value исходное текстовое значение
     * @return число либо null, если значение отсутствует или число не найдено
     */
    fun parseFirst(value: String?): BigDecimal? {
        if (value.isNullOrBlank()) {
            return null
        }

        return NUMBER_REGEX
            .find(value)
            ?.value
            ?.replace(",", ".")
            ?.toBigDecimalOrNull()
    }
}

/**
 * Содержит справочные данные Jira-импорта,
 * подготовленные один раз на весь запуск scheduler.
 *
 * Справочники индексируются по значениям, используемым
 * при сопоставлении Jira-данных с сущностями Пульта.
 */
data class JiraImportReferenceData(
    val strategiesByJiraKey: Map<String, StrategyEntity>,
    val enablersByNormalizedName: Map<String, EnablerEntity>,
    val statusesByCode: Map<String, StatusEntity>,
    val qualityGates: List<QualityGateEntity>,
    val divisionsByLabel: Map<String, DivisionEntity>,
    val blocksByLabel: Map<String, BlockEntity>,
    val initiativeTypesByCode: Map<String, InitiativeTypeEntity>,
)

/**
 * Загружает и подготавливает справочные данные,
 * используемые при импорте инициатив из Jira.
 *
 * Все данные читаются один раз на запуск scheduler,
 * чтобы исключить повторные обращения к БД
 * при обработке каждой инициативы.
 */
@Component
class JiraImportReferenceDataProvider(
    private val strategyService: StrategyService,
    private val enablerRepository: EnablerRepository,
    private val statusRepository: StatusRepository,
    private val qualityGateRepository: QualityGateRepository,
    private val divisionRepository: DivisionRepository,
    private val blockRepository: BlockRepository,
    private val initiativeTypeRepository: InitiativeTypeRepository,
    private val enablerNameNormalizer: EnablerNameNormalizer,
    private val jiraIssueKeyExtractor: JiraIssueKeyExtractor,
) {

    private val log by logger()

    /**
     * Загружает необходимые справочники и индексирует их
     * для последующего быстрого поиска.
     */
    fun load(): JiraImportReferenceData {
        val strategiesByJiraKey = strategyService
            .findAll()
            .mapNotNull { strategy ->
                jiraIssueKeyExtractor
                    .extract(strategy.jiraIssue)
                    ?.let { jiraKey -> jiraKey to strategy }
            }
            .toMap()

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

        val divisionsByLabel = divisionRepository
            .findAll()
            .mapNotNull { division ->
                division.label
                    ?.takeIf(String::isNotBlank)
                    ?.let { label -> label to division }
            }
            .toMap()

        val blocksByLabel = blockRepository
            .findAllByDisabledIsFalse()
            .mapNotNull { block ->
                block.label
                    ?.takeIf(String::isNotBlank)
                    ?.let { label -> label to block }
            }
            .toMap()

        val initiativeTypesByCode = initiativeTypeRepository
            .findAll()
            .mapNotNull { initiativeType ->
                initiativeType.code
                    ?.takeIf(String::isNotBlank)
                    ?.let { code -> code to initiativeType }
            }
            .toMap()

        log.debug(
            "Loaded Jira import reference data: " +
                "strategies={}, enablers={}, statuses={}, qualityGates={}, " +
                "divisions={}, blocks={}, initiativeTypes={}",
            strategiesByJiraKey.size,
            enablersByNormalizedName.size,
            statusesByCode.size,
            qualityGates.size,
            divisionsByLabel.size,
            blocksByLabel.size,
            initiativeTypesByCode.size,
        )

        return JiraImportReferenceData(
            strategiesByJiraKey = strategiesByJiraKey,
            enablersByNormalizedName = enablersByNormalizedName,
            statusesByCode = statusesByCode,
            qualityGates = qualityGates,
            divisionsByLabel = divisionsByLabel,
            blocksByLabel = blocksByLabel,
            initiativeTypesByCode = initiativeTypesByCode,
        )
    }
}

/**
 * Результат определения организационной принадлежности
 * Jira-инициативы.
 */
data class JiraInitiativeOrganization(
    val division: DivisionEntity?,
    val block: BlockEntity?,
)

/**
 * Определяет блок и трайб Jira-инициативы
 * по подразделению исполнителя и инициатора.
 *
 * При отсутствии подходящего значения блок и трайб
 * остаются незаполненными. Автоматический fallback на КИБ
 * намеренно не применяется.
 */
@Component
class JiraInitiativeOrganizationResolver {

    /**
     * Определяет block/division согласно приоритету FR1:
     *
     * 1. customfield_30001 -> division
     * 2. customfield_30000 -> division
     * 3. customfield_30001 -> block
     * 4. customfield_30000 -> block
     */
    fun resolveOrganization(
        initiatorUnits: List<String>?,
        executorUnits: List<String>?,
        referenceData: JiraImportReferenceData,
    ): JiraInitiativeOrganization {

        findDivision(
            values = executorUnits,
            divisionsByLabel = referenceData.divisionsByLabel,
        )?.let { division ->
            return JiraInitiativeOrganization(
                division = division,
                block = division.block,
            )
        }

        findDivision(
            values = initiatorUnits,
            divisionsByLabel = referenceData.divisionsByLabel,
        )?.let { division ->
            return JiraInitiativeOrganization(
                division = division,
                block = division.block,
            )
        }

        findBlock(
            values = executorUnits,
            blocksByLabel = referenceData.blocksByLabel,
        )?.let { block ->
            return JiraInitiativeOrganization(
                division = null,
                block = block,
            )
        }

        findBlock(
            values = initiatorUnits,
            blocksByLabel = referenceData.blocksByLabel,
        )?.let { block ->
            return JiraInitiativeOrganization(
                division = null,
                block = block,
            )
        }

        return JiraInitiativeOrganization(
            division = null,
            block = null,
        )
    }

    private fun findDivision(
        values: List<String>?,
        divisionsByLabel: Map<String, DivisionEntity>,
    ): DivisionEntity? {
        return values
            .orEmpty()
            .firstNotNullOfOrNull { value ->
                divisionsByLabel[value]
            }
    }

    private fun findBlock(
        values: List<String>?,
        blocksByLabel: Map<String, BlockEntity>,
    ): BlockEntity? {
        return values
            .orEmpty()
            .firstNotNullOfOrNull { value ->
                blocksByLabel[value]
            }
    }
}

/**
 * Определяет тип инициативы Пульта по Jira labels.
 */
@Component
class JiraInitiativeTypeResolver {

    companion object {
        private const val AGENT_LABEL_PART = "агент"
        private const val AGENT_TYPE_CODE = "agent"
        private const val GENERATIVE_AI_SOLUTION_TYPE_CODE = "genAiSolution"
    }

    /**
     * Если хотя бы одна Jira label содержит слово "агент"
     * без учёта регистра, возвращает тип agent.
     *
     * В остальных случаях возвращает genAiSolution.
     */
    fun resolveInitiativeType(
        labels: List<String>?,
        initiativeTypesByCode: Map<String, InitiativeTypeEntity>,
    ): InitiativeTypeEntity {
        val initiativeTypeCode = if (
            labels.orEmpty().any { label ->
                label.contains(
                    other = AGENT_LABEL_PART,
                    ignoreCase = true,
                )
            }
        ) {
            AGENT_TYPE_CODE
        } else {
            GENERATIVE_AI_SOLUTION_TYPE_CODE
        }

        return checkNotNull(
            initiativeTypesByCode[initiativeTypeCode]
        ) {
            "Initiative type '$initiativeTypeCode' is not configured"
        }
    }
}

/**
 * Определяет стратегии, связанные с Jira-инициативой.
 *
 * Сопоставление выполняется по Jira key из inward/outward issue links
 * и Jira key стратегии из справочника Пульта.
 */
@Component
class JiraStrategyResolver(
    private val jiraIssueKeyExtractor: JiraIssueKeyExtractor,
) {

    /**
     * Возвращает уникальные стратегии, связанные с инициативой.
     */
    fun resolveStrategies(
        issue: SearchIssueDto,
        strategiesByJiraKey: Map<String, StrategyEntity>,
    ): List<StrategyEntity> {
        val linkedJiraKeys = issue.fields
            ?.issuelinks
            .orEmpty()
            .asSequence()
            .flatMap { issueLink ->
                sequenceOf(
                    issueLink.outwardIssue?.key,
                    issueLink.inwardIssue?.key,
                )
            }
            .mapNotNull(jiraIssueKeyExtractor::extract)
            .toSet()

        return linkedJiraKeys
            .mapNotNull(strategiesByJiraKey::get)
            .distinctBy { strategy -> strategy.id }
    }
}

/**
 * Данные одного задействованного ресурса,
 * полученные из Jira.
 */
data class JiraInvolvedResourceData(
    val source: String,
    val type: String,
    val value: BigDecimal,
)

/**
 * Преобразует Jira resource fields в данные involved_resource.
 */
@Component
class JiraInvolvedResourceResolver(
    private val numericValueParser: JiraNumericValueParser,
) {

    private val log by logger()

    /**
     * Обрабатывает четыре Jira-поля ресурсов:
     *
     * customfield_31304 -> without_steerCo / business
     * customfield_31305 -> without_steerCo / it
     * customfield_31306 -> steerCo / business
     * customfield_31307 -> steerCo / it
     */
    fun resolveInvolvedResources(
        jiraKey: String,
        issue: SearchIssueDto,
    ): List<JiraInvolvedResourceData> {
        val fields = issue.fields ?: return emptyList()

        return listOfNotNull(
            resolveResource(
                jiraKey = jiraKey,
                jiraFieldName = "customfield_31304",
                jiraFieldValue = fields.customfield_31304,
                source = "without_steerCo",
                type = "business",
            ),
            resolveResource(
                jiraKey = jiraKey,
                jiraFieldName = "customfield_31305",
                jiraFieldValue = fields.customfield_31305,
                source = "without_steerCo",
                type = "it",
            ),
            resolveResource(
                jiraKey = jiraKey,
                jiraFieldName = "customfield_31306",
                jiraFieldValue = fields.customfield_31306,
                source = "steerCo",
                type = "business",
            ),
            resolveResource(
                jiraKey = jiraKey,
                jiraFieldName = "customfield_31307",
                jiraFieldValue = fields.customfield_31307,
                source = "steerCo",
                type = "it",
            ),
        )
    }

    private fun resolveResource(
        jiraKey: String,
        jiraFieldName: String,
        jiraFieldValue: String?,
        source: String,
        type: String,
    ): JiraInvolvedResourceData? {
        if (jiraFieldValue.isNullOrBlank()) {
            return null
        }

        val numericValue = numericValueParser.parseFirst(jiraFieldValue)

        if (numericValue == null) {
            log.warn(
                "Cannot parse involved resource from Jira: jiraKey={}, field={}, value={}",
                jiraKey,
                jiraFieldName,
                jiraFieldValue,
            )

            return null
        }

        return JiraInvolvedResourceData(
            source = source,
            type = type,
            value = numericValue,
        )
    }
}


/**
 * Создаёт новую инициативу Пульта на основании данных Jira.
 *
 * В рамках одной транзакции:
 * 1. создаёт AIAgentEntity;
 * 2. создаёт связи со стратегиями;
 * 3. сохраняет задействованные ресурсы.
 *
 * Каждая Jira-инициатива обрабатывается в отдельной транзакции,
 * чтобы ошибка одной инициативы не откатывала обработку остальных.
 */
@Service
class JiraNewInitiativeCreator(
    private val agentRepository: AIAgentRepository,
    private val agentStrategyRepository: AgentStrategyRepository,
    private val involvedResourceRepository: InvolvedResourceRepository,
    private val organizationResolver: JiraInitiativeOrganizationResolver,
    private val initiativeTypeResolver: JiraInitiativeTypeResolver,
    private val numericValueParser: JiraNumericValueParser,
    private val strategyResolver: JiraStrategyResolver,
    private val involvedResourceResolver: JiraInvolvedResourceResolver,
) {

    companion object {
        private const val ANALYSIS_STATUS_CODE = "analysis"

        private const val IMPORT_STATUS_BLOCKED = "blocked"
        private const val JIRA_FROM_STATUS_IN_PROGRESS = "inProgress"
        private const val JIRA_LINK_DONE = "done"

        private const val MAX_AGENT_NAME_LENGTH = 255
        private const val MAX_AGENT_DESCRIPTION_LENGTH = 2000

        private val log by logger()
    }

    /**
     * Создаёт одну новую Jira-инициативу и связанные с ней
     * данные второго этапа FR1.
     *
     * @param issue исходная Jira-инициатива
     * @param referenceData справочники текущего запуска scheduler
     */
    @Transactional(
        propagation = Propagation.REQUIRES_NEW,
        rollbackFor = [Exception::class],
    )
    fun createInitiativeFromJira(
        issue: SearchIssueDto,
        referenceData: JiraImportReferenceData,
    ) {
        val jiraKey = requireNotNull(issue.key) {
            "Jira initiative key must not be null"
        }

        log.debug(
            "Started creating new Jira initiative in Pult: jiraKey={}",
            jiraKey,
        )

        val analysisStatus = checkNotNull(
            referenceData.statusesByCode[ANALYSIS_STATUS_CODE]
        ) {
            "Active status '$ANALYSIS_STATUS_CODE' is not configured"
        }

        val organization = organizationResolver.resolve(
            initiatorUnits = issue.fields?.customfield_30000,
            executorUnits = issue.fields?.customfield_30001,
            referenceData = referenceData,
        )

        if (organization.block == null && organization.division == null) {
            log.debug(
                "Block and division were not resolved for new Jira initiative: " +
                    "jiraKey={}, customfield_30000={}, customfield_30001={}",
                jiraKey,
                issue.fields?.customfield_30000,
                issue.fields?.customfield_30001,
            )
        }

        val initiativeType = initiativeTypeResolver.resolve(
            labels = issue.fields?.labels,
            initiativeTypesByCode = referenceData.initiativeTypesByCode,
        )

        val optimizationEffect = parseEffect(
            jiraKey = jiraKey,
            jiraFieldName = "customfield_34300",
            jiraFieldValue = issue.fields?.customfield_34300,
        )

        val revenueEffect = parseEffect(
            jiraKey = jiraKey,
            jiraFieldName = "customfield_30401",
            jiraFieldValue = issue.fields?.customfield_30401,
        )

        val currentDateTime = LocalDateTime.now()

        val agent = AIAgentEntity(
            agentId = jiraKey,
            agentName = truncate(
                jiraKey = jiraKey,
                jiraFieldName = "summary",
                value = issue.fields?.summary,
                maxLength = MAX_AGENT_NAME_LENGTH,
            ),
            agentDescription = truncate(
                jiraKey = jiraKey,
                jiraFieldName = "description",
                value = issue.fields?.description,
                maxLength = MAX_AGENT_DESCRIPTION_LENGTH,
            ),
            agentJiraUrl = jiraKey,
            agentEffectOptimization = optimizationEffect,
            agentEffectRevenue = revenueEffect,
            initiativeType = initiativeType,
            block = organization.block,
            division = organization.division,
            agentStatus = analysisStatus,
            importStatus = IMPORT_STATUS_BLOCKED,
            jiraFromStatus = JIRA_FROM_STATUS_IN_PROGRESS,
        ).apply {
            created = currentDateTime
        }

        val savedAgent = agentRepository.save(agent)

        log.debug(
            "Created ai_agent from Jira: jiraKey={}, agentId={}, block={}, division={}, initiativeType={}",
            jiraKey,
            savedAgent.id,
            savedAgent.block?.code,
            savedAgent.division?.code,
            savedAgent.initiativeType?.code,
        )

        createStrategies(
            agent = savedAgent,
            issue = issue,
            referenceData = referenceData,
        )

        createInvolvedResources(
            agent = savedAgent,
            issue = issue,
            jiraKey = jiraKey,
            currentDateTime = currentDateTime,
        )

        log.info(
            "Successfully created new Jira initiative in Pult: jiraKey={}, agentId={}",
            jiraKey,
            savedAgent.id,
        )
    }

    /**
     * Создаёт связи новой инициативы со стратегиями.
     */
    private fun createStrategies(
        agent: AIAgentEntity,
        issue: SearchIssueDto,
        referenceData: JiraImportReferenceData,
    ) {
        val strategies = strategyResolver.resolve(
            issue = issue,
            strategiesByJiraKey = referenceData.strategiesByJiraKey,
        )

        if (strategies.isEmpty()) {
            log.debug(
                "No strategies found for Jira initiative: jiraKey={}",
                issue.key,
            )
            return
        }

        val agentStrategies = strategies.map { strategy ->
            AgentStrategyEntity(
                agent = agent,
                strategy = strategy,
                jiraLink = JIRA_LINK_DONE,
            )
        }

        agentStrategyRepository.saveAll(agentStrategies)

        log.debug(
            "Created agent strategies: jiraKey={}, agentId={}, strategyIds={}",
            issue.key,
            agent.id,
            strategies.map { strategy -> strategy.id },
        )
    }

    /**
     * Создаёт записи задействованных ресурсов новой инициативы.
     */
    private fun createInvolvedResources(
        agent: AIAgentEntity,
        issue: SearchIssueDto,
        jiraKey: String,
        currentDateTime: LocalDateTime,
    ) {
        val resources = involvedResourceResolver.resolve(
            jiraKey = jiraKey,
            issue = issue,
        )

        if (resources.isEmpty()) {
            log.debug(
                "No involved resources found for Jira initiative: jiraKey={}",
                jiraKey,
            )
            return
        }

        val involvedResourceEntities = resources.map { resource ->
            InvolvedResourceEntity().apply {
                id = InvolvedResourceEmbeddedId(
                    aiAgentId = agent.id,
                    source = resource.source,
                    type = resource.type,
                )

                value = resource.value
                timeAllocated = null

                created = currentDateTime
                updated = currentDateTime

                aiAgent = agent
            }
        }

        involvedResourceRepository.saveAll(involvedResourceEntities)

        log.debug(
            "Created involved resources: jiraKey={}, agentId={}, resources={}",
            jiraKey,
            agent.id,
            resources.map { resource ->
                "${resource.source}/${resource.type}=${resource.value}"
            },
        )
    }

    /**
     * Извлекает числовое значение эффекта из Jira-поля.
     *
     * Некорректное непустое значение не блокирует создание инициативы,
     * но фиксируется в логах.
     */
    private fun parseEffect(
        jiraKey: String,
        jiraFieldName: String,
        jiraFieldValue: String?,
    ): BigDecimal? {
        if (jiraFieldValue.isNullOrBlank()) {
            return null
        }

        val numericValue = numericValueParser.parseFirst(jiraFieldValue)

        if (numericValue == null) {
            log.warn(
                "Cannot parse initiative effect from Jira: jiraKey={}, field={}, value={}",
                jiraKey,
                jiraFieldName,
                jiraFieldValue,
            )
        }

        return numericValue
    }

    /**
     * Ограничивает строковое значение максимально допустимой длиной.
     *
     * Если значение было обрезано, это фиксируется в DEBUG-логе.
     */
    private fun truncate(
        jiraKey: String,
        jiraFieldName: String,
        value: String?,
        maxLength: Int,
    ): String? {
        if (value == null || value.length <= maxLength) {
            return value
        }

        log.debug(
            "Jira field was truncated while creating initiative: " +
                "jiraKey={}, field={}, originalLength={}, maxLength={}",
            jiraKey,
            jiraFieldName,
            value.length,
            maxLength,
        )

        return value.take(maxLength)
    }
}

/**
 * Содержит статистику обработки одной страницы Jira Search.
 */
data class JiraInitiativePageStatistics(
    val existingInitiatives: Int = 0,
    val newInitiatives: Int = 0,
    val createdInitiatives: Int = 0,
    val ignoredInitiatives: Int = 0,
    val failedInitiatives: Int = 0,
)

/**
 * Оркестрирует поиск и создание новых Jira-инициатив в рамках FR1.
 *
 * Получает настройки и справочники, выполняет постраничный Jira Search,
 * применяет ограничения FR1, проверяет существование инициатив
 * и передаёт новые инициативы в отдельный транзакционный creator.
 */
@Service
class JiraNewInitiativeSearchSchedulerService(
    private val optionsService: OptionsService,
    private val referenceDataProvider: JiraImportReferenceDataProvider,
    private val searchRequestFactory: JiraInitiativeSearchRequestFactory,
    private val jiraSearchPaginator: JiraSearchPaginator,
    private val existingJiraInitiativeRepository: ExistingJiraInitiativeRepository,
    private val jiraNewInitiativeCreator: JiraNewInitiativeCreator,
) {

    companion object {
        private const val CANCELLED_STATUS_NAME = "Отменена"
        private const val CLASSIC_ML_LABEL = "ClassicML"
        private const val CROSSGOAL_KEY_PREFIX = "CROSSGOAL-"

        private val log by logger()
    }

    /**
     * Выполняет поиск и создание новых инициатив из Jira.
     *
     * Справочники загружаются один раз на запуск.
     * Jira-результаты обрабатываются постранично без накопления
     * всех инициатив в памяти.
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
            "Jira reference data prepared: strategies={}, enablers={}, statuses={}, " +
                "qualityGates={}, divisions={}, blocks={}, initiativeTypes={}",
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
     * Сначала исключает неподходящие инициативы,
     * затем одним batch-запросом проверяет существование оставшихся ключей.
     * Новые инициативы создаются независимо друг от друга.
     */
    private fun processPage(
        issues: List<SearchIssueDto>,
        referenceData: JiraImportReferenceData,
    ): JiraInitiativePageStatistics {
        if (issues.isEmpty()) {
            return JiraInitiativePageStatistics()
        }

        var ignoredInitiatives = 0

        val initiativesByJiraKey = buildMap<String, SearchIssueDto> {
            issues.forEach { issue ->
                val normalizedJiraKey = normalizeJiraKey(issue.key)

                if (normalizedJiraKey == null) {
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

                put(normalizedJiraKey, issue)
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
                jiraNewInitiativeCreator.create(
                    issue = issue,
                    referenceData = referenceData,
                )

                createdInitiatives++
            } catch (exception: Exception) {
                failedInitiatives++

                log.error(
                    "Failed to create new Jira initiative: jiraKey={}, error={}",
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

    /**
     * Нормализует Jira key исключительно для технического
     * поиска существующей инициативы.
     *
     * При сохранении новой инициативы используется исходный issue.key,
     * поскольку FR1 требует сохранять Jira key без изменений.
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



```
