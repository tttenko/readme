```java
==============================================================================
Дополнения в репозитории
RepositoryAdditions.kt.txt
==============================================================================
// Добавить внутрь существующего AIAgentRepository:

@Query("""
    SELECT initiative.id AS initiativeId,
           process.shortName AS processCode
    FROM AIAgentEntity initiative
    JOIN initiative.processes process
    WHERE initiative.id IN :initiativeIds
""")
fun findBusinessProcessCodesForRegistryExport(
    @Param("initiativeIds") initiativeIds: Collection<Long>
): List<InitiativeProcessCodeExportProjection>

// Добавить внутрь существующего AgentContactRepository:

@Query("""
    SELECT agentContact.agent.id AS initiativeId,
           agentContact.type AS contactType,
           contact.email AS email,
           agentContact.userId AS userId
    FROM AgentContactEntity agentContact
    LEFT JOIN agentContact.contact contact
    WHERE agentContact.agent.id IN :initiativeIds
""")
fun findContactsForRegistryExport(
    @Param("initiativeIds") initiativeIds: Collection<Long>
): List<InitiativeContactExportProjection>

// Отдельные интерфейсы проекций в repository-пакете:

interface InitiativeProcessCodeExportProjection {
    val initiativeId: Long
    val processCode: String?
}

interface InitiativeContactExportProjection {
    val initiativeId: Long
    val contactType: String?
    val email: String?
    val userId: Long?
}



==============================================================================
Модель экспорта
InitiativeRegistryExcelExportModel.kt
==============================================================================
/** Строка первого листа выгрузки Пульт AI-инициатив. */
data class InitiativeRegistryExcelExportModel(
    val id: String?,
    val block: String?,
    val division: String?,
    val terBank: String?,
    val name: String?,
    val description: String?,
    val problem: String?,
    val status: String?,
    val jiraUrl: String?,
    val crossgoal: String?,
    val gigausage: String?,
    val initiativeType: String?,
    val audience: String?,
    val program: String?,
    val effectRevenue: BigDecimal?,
    val effectOptimization: BigDecimal?,
    val implementedPlatforms: String?,
    val isDisable: Boolean?,
    val initiativeUrl: String?,
    val businessProcesses: String?,
    val businessOwner: String?,
    val itOwner: String?,
    val stageStatusConcept: String?,
    val stageDeadlineConcept: String?,
    val stageStatusPoc: String?,
    val stageDeadlinePoc: String?,
    val stageStatusMvp: String?,
    val stageDeadlineMvp: String?,
    val stageStatusTargetSolution: String?,
    val stageDeadlineTargetSolution: String?,
    val ciSolution: String?
) {
    companion object {
        const val STAGE_COMPLETED = "Завершён"
        const val STAGE_IN_PROGRESS = "В работе"
        const val STAGE_PLANNED = "План"

        const val STAGE_ANALYSIS = "analysis"
        const val STAGE_DEVELOPMENT = "development"
        const val STAGE_PILOT = "pilot"
        const val STAGE_TARGET_SOLUTION = "targetSolution"
    }
}



==============================================================================
Полный Mapper
InitiativeRegistryMapper.kt
==============================================================================
@Component
class InitiativeRegistryMapper {

    companion object {
        private val EXCEL_DATE_FORMATTER = DateTimeFormatter.ofPattern("dd.MM.yyyy")
    }

    /**
     * Приводит сырой код текущего статуса инициативы к «витринному».
     * Служебные статусы нормализуются, как на фронтенде:
     * `research` -> `analysis`, `release` -> `targetSolution` (BR-005, technical.md).
     */
    fun normalizeShowcaseStatusCode(code: String?): String? = when (code) {
        "research" -> InitiativeRegistryExcelExportModel.STAGE_ANALYSIS
        "release" -> InitiativeRegistryExcelExportModel.STAGE_TARGET_SOLUTION
        else -> code
    }

    /**
     * Жизненный статус этапа (FR-029):
     * completedDate != null -> `Завершён`;
     * иначе этап раньше витринного статуса -> `Завершён`;
     * этап равен витринному статусу -> `В работе`;
     * этап после витринного статуса -> `План`.
     */
    fun computeStageStatus(
        sla: AgentStatusSlaEntity?,
        stageOrdering: Long?,
        currentShowcaseOrdering: Long?,
    ): String? {
        if (sla?.completedDate != null) return InitiativeRegistryExcelExportModel.STAGE_COMPLETED
        if (stageOrdering == null) return InitiativeRegistryExcelExportModel.STAGE_PLANNED
        if (currentShowcaseOrdering == null) return InitiativeRegistryExcelExportModel.STAGE_PLANNED
        return when {
            stageOrdering < currentShowcaseOrdering -> InitiativeRegistryExcelExportModel.STAGE_COMPLETED
            stageOrdering == currentShowcaseOrdering -> InitiativeRegistryExcelExportModel.STAGE_IN_PROGRESS
            else -> InitiativeRegistryExcelExportModel.STAGE_PLANNED
        }
    }

    /**
     * Дедлайн этапа (FR-030): plannedDate; при отсутствии — пустая ячейка.
     */
    fun computeStageDeadline(sla: AgentStatusSlaEntity?): String? =
        sla?.plannedDate?.toLocalDate()?.format(EXCEL_DATE_FORMATTER)

    fun toInitiativeRegistryExcelExportModel(
        entity: AIAgentEntity,
        terBanks: List<TerBank> = emptyList(),
        deltaPrefix: String = "",
        sigmaPrefix: String = "",
        statusOrderingByCode: Map<String, Long> = emptyMap(),
        slaByStatusCode: Map<String, AgentStatusSlaEntity> = emptyMap(),
        urlTemplate: String = "",
        userEmailsById: Map<Long, String> = emptyMap(),
        contacts: List<InitiativeContactExportProjection> = emptyList(),
        businessProcessCodes: List<String> = emptyList()
    ): InitiativeRegistryExcelExportModel {
        val base = entity.toAIAgentExcelExportModel(terBanks, deltaPrefix, sigmaPrefix)

        val currentShowcaseOrdering = normalizeShowcaseStatusCode(entity.agentStatus?.code)
            ?.let { statusOrderingByCode[it] }

        return InitiativeRegistryExcelExportModel(
            id = base.id,
            block = base.block,
            division = base.division,
            terBank = base.terBank,
            name = base.name,
            description = base.description,
            problem = base.problem,
            status = base.status,
            jiraUrl = base.jiraUrl,
            crossgoal = base.crossgoal,
            gigausage = base.gigausage,
            initiativeType = base.initiativeType,
            audience = base.audience,
            program = base.program,
            effectRevenue = base.effectRevenue,
            effectOptimization = base.effectOptimization,
            implementedPlatforms = base.implementedPlatforms,
            isDisable = entity.disabled,
            initiativeUrl = if (urlTemplate.isBlank()) null else urlTemplate.replace("{id}", entity.id.toString()),
            businessProcesses = businessProcessCodes.joinToString(";"),
            businessOwner = collectEmailsByRole(contacts, "customer", userEmailsById),
            itOwner = collectEmailsByRole(contacts, "developer", userEmailsById),
            stageStatusConcept = computeStageStatus(
                slaByStatusCode[InitiativeRegistryExcelExportModel.STAGE_ANALYSIS],
                statusOrderingByCode[InitiativeRegistryExcelExportModel.STAGE_ANALYSIS],
                currentShowcaseOrdering,
            ),
            stageDeadlineConcept = computeStageDeadline(
                slaByStatusCode[InitiativeRegistryExcelExportModel.STAGE_ANALYSIS],
            ),
            stageStatusPoc = computeStageStatus(
                slaByStatusCode[InitiativeRegistryExcelExportModel.STAGE_DEVELOPMENT],
                statusOrderingByCode[InitiativeRegistryExcelExportModel.STAGE_DEVELOPMENT],
                currentShowcaseOrdering,
            ),
            stageDeadlinePoc = computeStageDeadline(
                slaByStatusCode[InitiativeRegistryExcelExportModel.STAGE_DEVELOPMENT],
            ),
            stageStatusMvp = computeStageStatus(
                slaByStatusCode[InitiativeRegistryExcelExportModel.STAGE_PILOT],
                statusOrderingByCode[InitiativeRegistryExcelExportModel.STAGE_PILOT],
                currentShowcaseOrdering,
            ),
            stageDeadlineMvp = computeStageDeadline(
                slaByStatusCode[InitiativeRegistryExcelExportModel.STAGE_PILOT],
            ),
            stageStatusTargetSolution = computeStageStatus(
                slaByStatusCode[InitiativeRegistryExcelExportModel.STAGE_TARGET_SOLUTION],
                statusOrderingByCode[InitiativeRegistryExcelExportModel.STAGE_TARGET_SOLUTION],
                currentShowcaseOrdering,
            ),
            stageDeadlineTargetSolution = computeStageDeadline(
                slaByStatusCode[InitiativeRegistryExcelExportModel.STAGE_TARGET_SOLUTION],
            ),
            ciSolution = entity.ciSolutions.filter { it.isActive }.mapNotNull { it.ciSolution }.joinToString(";")
        )
    }
    /**
     * Дедупликация выполняется только в рамках указанного типа контакта.
     * Один email может присутствовать в customer и developer одновременно.
     */
    private fun collectEmailsByRole(
        contacts: List<InitiativeContactExportProjection>,
        role: String,
        userEmailsById: Map<Long, String>
    ): String = contacts.asSequence()
        .filter { it.contactType == role }
        .mapNotNull { contact ->
            val emailFromContact = contact.email?.trim()?.takeIf { it.isNotBlank() }
            val emailFromUser = contact.userId?.let { userEmailsById[it] }
            emailFromContact ?: emailFromUser
        }
        .distinct()
        .joinToString(";")

}



==============================================================================
Полный Service
InitiativeRegistryExportService.kt
==============================================================================
@Service
class InitiativeRegistryExportService(
    private val mapper: InitiativeRegistryMapper,
    private val terBankService: TerBankService,
    private val aiAgentRepository: AIAgentRepository,
    private val statusRepository: StatusRepository,
    private val agentStatusSlaRepository: AgentStatusSlaRepository,
    private val authFeignClient: PrmAuthFeignClient,
    private val financialEffectRepository: InitiativeFinancialEffectRepository,
    private val agentContactRepository: AgentContactRepository
) {

    @Value("\${scheduled.jira-sync.delta.url.prefix}")
    private var deltaPrefix: String = ""

    @Value("\${scheduled.jira-sync.sigma.url.prefix}")
    private var sigmaPrefix: String = ""

    @Value("\${prm.email.link.link-to-portal-short}")
    private var portalUrl: String = ""

    companion object {
        const val XLSX_MEDIA_TYPE_VALUE = "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"
        val XLSX_MEDIA_TYPE: MediaType = MediaType.parseMediaType(XLSX_MEDIA_TYPE_VALUE)

        const val FILE_NAME = "Портфель AI-инициатив.xlsx"

        private const val SHEET_NAME = "Портфель AI-инициатив"
        private const val FINANCIAL_EFFECT_SHEET_NAME = "Финансовый эффект"

        private val FINANCIAL_EFFECT_DATE_TIME_FORMATTER = DateTimeFormatter.ofPattern("dd.MM.yyyy HH:mm:ss")
    }

    /**
     * Формирует XLSX-реестр AI-инициатив с двумя листами.
     *
     * Первый лист содержит существующие данные инициатив без старых
     * столбцов финансового эффекта.
     *
     * Второй лист содержит все активные финансовые эффекты
     * выгружаемых инициатив за все годы.
     */
    @Transactional(readOnly = true)
    fun downloadRegistryExcel(): ResponseEntity<InputStreamResource> {
        val urlTemplate = buildInitiativeUrlTemplate(portalUrl)

        val agents = aiAgentRepository.findAll()
        val initiativeIds = agents.map { it.id }

        val contactsByInitiativeId = loadContactsByInitiativeId(initiativeIds)
        val businessProcessCodesByInitiativeId = loadBusinessProcessCodesByInitiativeId(initiativeIds)
        val userEmailsById = loadUserEmailsById(contactsByInitiativeId.values.flatten())
        val terBanks = terBankService.list()
        val statusOrderingByCode = loadStatusOrderingByCode()
        val slaByInitiativeAndStatusCode = loadSlaByInitiativeAndStatusCode(initiativeIds)

        val initiativeRows = agents.map { agent ->
            mapper.toInitiativeRegistryExcelExportModel(
                entity = agent,
                terBanks = terBanks,
                deltaPrefix = deltaPrefix,
                sigmaPrefix = sigmaPrefix,
                statusOrderingByCode = statusOrderingByCode,
                slaByStatusCode = slaByInitiativeAndStatusCode[agent.id].orEmpty(),
                urlTemplate = urlTemplate,
                userEmailsById = userEmailsById,
                contacts = contactsByInitiativeId[agent.id].orEmpty(),
                businessProcessCodes = businessProcessCodesByInitiativeId[agent.id].orEmpty()
            )
        }

        val financialEffectRows = loadFinancialEffectRows(initiativeIds)

        val workbook = ExcelExportHelper.createWorkBook(
            listOf(SHEET_NAME, FINANCIAL_EFFECT_SHEET_NAME)
        )

        ExcelExportHelper.writeSheetData(
            workbook,
            workbook.getSheetAt(0),
            initiativeRows,
            headerColumns()
        )

        ExcelExportHelper.writeSheetData(
            workbook,
            workbook.getSheetAt(1),
            financialEffectRows,
            getFinancialEffectHeaderColumns()
        )

        return with(ExcelExportHelper) {
            workbook.convertToFile(FILE_NAME, XLSX_MEDIA_TYPE)
        }
    }

    /**
     * Загружает активные финансовые эффекты всех выгружаемых инициатив.
     *
     * Используется один запрос к БД независимо от количества инициатив.
     * Для каждой карточки формируется отдельная строка второго листа.
     */
    private fun loadFinancialEffectRows(
        initiativeIds: Collection<Long>
    ): List<InitiativeFinancialEffectExcelExportModel> {
        if (initiativeIds.isEmpty()) return emptyList()

        return financialEffectRepository.findActiveByInitiativeIds(initiativeIds)
            .map { financialEffect -> buildFinancialEffectExcelExportModel(financialEffect) }
    }

    /**
     * Преобразует карточку финансового эффекта в строку Excel.
     */
    private fun buildFinancialEffectExcelExportModel(
        financialEffect: InitiativeFinancialEffectEntity
    ): InitiativeFinancialEffectExcelExportModel {
        val financialEffectType = financialEffect.type!!

        return InitiativeFinancialEffectExcelExportModel(
            initiativeId = financialEffect.initiative!!.id.toString(),
            financialEffectId = financialEffect.id.toString(),
            effectYear = financialEffect.effectYear,
            typeCode = financialEffectType.code,
            typeName = financialEffectType.name,
            unit = financialEffectType.unit,
            planValue = financialEffect.planValue,
            factValue = financialEffect.factValue,
            targetMonth = financialEffect.targetMonth?.let { YearMonth.from(it).toString() },
            approvalLinks = financialEffect.approvalLinks?.joinToString("; ") { it.asText() }.orEmpty(),
            createdAt = financialEffect.created?.format(FINANCIAL_EFFECT_DATE_TIME_FORMATTER)
        )
    }

    /**
     * Описывает столбцы листа «Финансовый эффект».
     *
     * План и факт записываются как числа.
     * Отсутствующий факт оставляет ячейку пустой.
     */
    private fun getFinancialEffectHeaderColumns(): List<ExcelColumnDescription<InitiativeFinancialEffectExcelExportModel>> =
        listOf(
            ExcelExportHelper.textColumn("ID инициативы") { it.initiativeId },
            ExcelExportHelper.textColumn("ID финансового эффекта") { it.financialEffectId },
            ExcelExportHelper.numberColumn("Год финансового эффекта") { it.effectYear },
            ExcelExportHelper.textColumn("Код типа") { it.typeCode },
            ExcelExportHelper.textColumn("Тип финансового эффекта") { it.typeName },
            ExcelExportHelper.textColumn("Единица измерения") { it.unit },
            ExcelExportHelper.numberColumn("План") { it.planValue },
            ExcelExportHelper.numberColumn("Факт") { it.factValue },
            ExcelExportHelper.textColumn("Срок достижения") { it.targetMonth },
            ExcelExportHelper.textColumn("Ссылки на согласование") { it.approvalLinks },
            ExcelExportHelper.textColumn("Дата добавления") { it.createdAt }
        )

    /**
     * Получает контактные связи инициатив за один запрос.
     */
    private fun loadContactsByInitiativeId(
        initiativeIds: Collection<Long>
    ): Map<Long, List<InitiativeContactExportProjection>> {
        if (initiativeIds.isEmpty()) return emptyMap()

        return agentContactRepository.findContactsForRegistryExport(initiativeIds)
            .groupBy { it.initiativeId }
    }

    /**
     * Получает коды бизнес-процессов (ProcessEntity.shortName) за один запрос.
     */
    private fun loadBusinessProcessCodesByInitiativeId(
        initiativeIds: Collection<Long>
    ): Map<Long, List<String>> {
        if (initiativeIds.isEmpty()) return emptyMap()

        return aiAgentRepository.findBusinessProcessCodesForRegistryExport(initiativeIds)
            .groupBy { it.initiativeId }
            .mapValues { (_, processRows) ->
                processRows.mapNotNull { process ->
                    process.processCode?.trim()?.takeIf { it.isNotBlank() }
                }.distinct().sorted()
            }
    }

    /**
     * Сохраняет поддержку внутренних контактов с userId без contact.email.
     * Email всех пользователей запрашиваются одним bulk-вызовом.
     */
    private fun loadUserEmailsById(
        contacts: Collection<InitiativeContactExportProjection>
    ): Map<Long, String> {
        val userIds = contacts.asSequence()
            .filter { it.email.isNullOrBlank() }
            .mapNotNull { it.userId }
            .toSet()

        if (userIds.isEmpty()) return emptyMap()

        return authFeignClient.getUsers(ids = userIds, companyId = null)
            .body
            .orEmpty()
            .mapNotNull { user ->
                val email = user.email?.trim()?.takeIf { it.isNotBlank() }
                    ?: return@mapNotNull null
                user.id to email
            }
            .toMap()
    }

    /**
     * Формирует шаблон ссылки на инициативу в Пульте.
     */
    private fun buildInitiativeUrlTemplate(portalUrl: String): String {
        if (portalUrl.isBlank()) {
            throw AiInternalServerException(
                message = "Property 'prm.email.link.link-to-portal-short' is not configured"
            )
        }

        return "${portalUrl.trimEnd('/')}/ai/initiatives/{id}"
    }

    /**
     * Загружает ordering активных статусов и индексирует их по code.
     */
    private fun loadStatusOrderingByCode(): Map<String, Long> =
        statusRepository.findAllActive()
            .mapNotNull { status ->
                val code = status.code ?: return@mapNotNull null
                val ordering = status.ordering ?: return@mapNotNull null
                code to ordering
            }
            .toMap()

    /**
     * Загружает SLA всех выгружаемых инициатив одним запросом.
     *
     * Результат группируется по ID инициативы и коду этапа.
     */
    private fun loadSlaByInitiativeAndStatusCode(
        initiativeIds: Collection<Long>
    ): Map<Long, Map<String, AgentStatusSlaEntity>> {
        if (initiativeIds.isEmpty()) return emptyMap()

        return agentStatusSlaRepository.findAllByAiAgentIdIn(initiativeIds)
            .mapNotNull { sla ->
                val initiativeId = sla.primaryKey.aiAgentId ?: return@mapNotNull null
                val statusCode = sla.agentStatus?.code ?: return@mapNotNull null
                initiativeId to (statusCode to sla)
            }
            .groupBy({ it.first }, { it.second })
            .mapValues { (_, pairs) -> pairs.toMap() }
    }

    /**
     * Описание 29 колонок первого листа XLSX.
     *
     * Старые колонки «Фин. эффект, руб.» и «Фин. эффект, ПШЕ»
     * полностью исключены из структуры листа.
     */
    fun headerColumns(): List<ExcelColumnDescription<InitiativeRegistryExcelExportModel>> =
        listOf(
            ExcelExportHelper.textColumn("ID AI-агента") { it.id },
            ExcelExportHelper.textColumn("Блок") { it.block },
            ExcelExportHelper.textColumn("Трайб") { it.division },
            ExcelExportHelper.textColumn("ТБ") { it.terBank },
            ExcelExportHelper.textColumn("Наименование") { it.name },
            ExcelExportHelper.textColumn("Описание") { it.description },
            ExcelExportHelper.textColumn("Проблема, которую решает") { it.problem },
            ExcelExportHelper.textColumn("Текущий статус") { it.status },
            ExcelExportHelper.textColumn("Ссылка JIRA") { it.jiraUrl },
            ExcelExportHelper.textColumn("CROSSGOAL") { it.crossgoal },
            ExcelExportHelper.textColumn("GIGAUSAGE") { it.gigausage },
            ExcelExportHelper.textColumn("Тип инициативы") { it.initiativeType },
            ExcelExportHelper.textColumn("Аудитория") { it.audience },
            ExcelExportHelper.textColumn("Программа AI-трансформации") { it.program },
            ExcelExportHelper.textColumn("Поверхности") { it.implementedPlatforms },
            ExcelExportHelper.textColumn("Архив") {
                when (it.isDisable) {
                    true -> "Да"
                    false -> "Нет"
                    null -> null
                }
            },
            ExcelExportHelper.hyperlinkColumn("Ссылка на инициативу в Пульт") { it.initiativeUrl },
            ExcelExportHelper.textColumn("Бизнес-процесс") { it.businessProcesses },
            ExcelExportHelper.textColumn("Бизнес-владелец") { it.businessOwner },
            ExcelExportHelper.textColumn("IT-владелец") { it.itOwner },
            ExcelExportHelper.textColumn("Статус этапа — Концепция") { it.stageStatusConcept },
            ExcelExportHelper.textColumn("Дедлайн этапа — Концепция") { it.stageDeadlineConcept },
            ExcelExportHelper.textColumn("Статус этапа — PoC") { it.stageStatusPoc },
            ExcelExportHelper.textColumn("Дедлайн этапа — PoC") { it.stageDeadlinePoc },
            ExcelExportHelper.textColumn("Статус этапа — MVP") { it.stageStatusMvp },
            ExcelExportHelper.textColumn("Дедлайн этапа — MVP") { it.stageDeadlineMvp },
            ExcelExportHelper.textColumn("Статус этапа — Целевое решение") { it.stageStatusTargetSolution },
            ExcelExportHelper.textColumn("Дедлайн этапа — Целевое решение") { it.stageDeadlineTargetSolution },
            ExcelExportHelper.textColumn("КЭ модуля") { it.ciSolution }
        )
}

```
