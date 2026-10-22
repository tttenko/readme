```java

@Service
class InitiativeRegistryExportService(
    private val mapper: InitiativeRegistryMapper,
    private val terBankService: TerBankService,
    private val aiAgentRepository: AIAgentRepository,
    private val statusRepository: StatusRepository,
    private val agentStatusSlaRepository: AgentStatusSlaRepository,
    private val authFeignClient: PrmAuthFeignClient,
    private val financialEffectRepository: InitiativeFinancialEffectRepository
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

        private val FINANCIAL_EFFECT_DATE_FORMATTER = DateTimeFormatter.ofPattern("dd.MM.yyyy")
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

        val userEmailsById = loadUserEmailsById(agents)
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
                userEmailsById = userEmailsById
            )
        }

        val initiativeRowsById = agents.zip(initiativeRows)
            .associate { (agent, initiativeRow) -> agent.id to initiativeRow }

        val financialEffectRows = loadFinancialEffectRows(initiativeRowsById)

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
     * Загружает активные финансовые эффекты выгружаемых инициатив.
     *
     * Для каждой карточки формируется отдельная строка Excel.
     * ID и наименование инициативы берутся из данных первого листа.
     */
    private fun loadFinancialEffectRows(
        initiativeRowsById: Map<Long, InitiativeRegistryExcelExportModel>
    ): List<InitiativeFinancialEffectExcelExportModel> {
        if (initiativeRowsById.isEmpty()) return emptyList()

        return financialEffectRepository.findActiveByInitiativeIds(initiativeRowsById.keys)
            .mapNotNull { financialEffect ->
                val initiativeId = financialEffect.initiative?.id ?: return@mapNotNull null
                val initiativeRow = initiativeRowsById[initiativeId] ?: return@mapNotNull null

                buildFinancialEffectExcelExportModel(financialEffect, initiativeRow)
            }
    }

    /**
     * Преобразует финансовый эффект в строку второго листа.
     *
     * Срок достижения отображается последним днём указанного месяца.
     */
    private fun buildFinancialEffectExcelExportModel(
        financialEffect: InitiativeFinancialEffectEntity,
        initiativeRow: InitiativeRegistryExcelExportModel
    ): InitiativeFinancialEffectExcelExportModel {
        return InitiativeFinancialEffectExcelExportModel(
            agentId = initiativeRow.id,
            agentName = initiativeRow.name,
            typeName = financialEffect.type?.name,
            effectYear = financialEffect.effectYear,
            targetMonth = financialEffect.targetMonth?.let { targetMonth ->
                targetMonth.withDayOfMonth(targetMonth.lengthOfMonth())
                    .format(FINANCIAL_EFFECT_DATE_FORMATTER)
            },
            planValue = financialEffect.planValue,
            factValue = financialEffect.factValue,
            approvalLinks = financialEffect.approvalLinks
                ?.joinToString("; ") { approvalLink -> approvalLink.asText() }
                .orEmpty()
        )
    }

    /**
     * Описывает восемь колонок листа «Финансовый эффект»
     * в соответствии с шаблоном Excel.
     */
    private fun getFinancialEffectHeaderColumns(): List<ExcelColumnDescription<InitiativeFinancialEffectExcelExportModel>> =
        listOf(
            ExcelExportHelper.textColumn("ID AI-агента") { it.agentId },
            ExcelExportHelper.textColumn("Наименование") { it.agentName },
            ExcelExportHelper.textColumn("Тип эффекта") { it.typeName },
            ExcelExportHelper.numberColumn("Год планирования") { it.effectYear },
            ExcelExportHelper.textColumn("Срок") { it.targetMonth },
            ExcelExportHelper.numberColumn("План") { it.planValue },
            ExcelExportHelper.numberColumn("Факт") { it.factValue },
            ExcelExportHelper.textColumn("Ссылки на jira") { it.approvalLinks }
        )

    /**
     * Загружает email пользователей АУЕ, указанных в agent_contact.user_id.
     *
     * Все уникальные userId запрашиваются одним bulk-вызовом.
     */
    private fun loadUserEmailsById(agents: List<AIAgentEntity>): Map<Long, String> {
        val userIds = agents.asSequence()
            .flatMap { it.agentContact.asSequence() }
            .mapNotNull { it.userId }
            .toSet()

        if (userIds.isEmpty()) return emptyMap()

        return authFeignClient.getUsers(ids = userIds, companyId = null)
            .body
            .orEmpty()
            .mapNotNull { user ->
                val email = user.email?.takeIf { it.isNotBlank() } ?: return@mapNotNull null
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
     * Описание 27 колонок первого листа XLSX.
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
            ExcelExportHelper.textColumn("Email контакта") { it.contactEmails },
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

data class InitiativeFinancialEffectExcelExportModel(
    val agentId: String?,
    val agentName: String?,
    val typeName: String?,
    val effectYear: Int,
    val targetMonth: String?,
    val planValue: BigDecimal?,
    val factValue: BigDecimal?,
    val approvalLinks: String
)
```
