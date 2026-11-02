```java

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

        const val FILE_NAME_TEMPLATE = "PULT_AI-инициативы_%s.xlsx"

        private const val SHEET_NAME = "Портфель AI-инициатив"
        private const val FINANCIAL_EFFECT_SHEET_NAME = "Финансовый эффект"

        private val FILE_NAME_TIMESTAMP_FORMATTER = DateTimeFormatter.ofPattern("ddMMyyyy_HHmmssSSS")
        private val FINANCIAL_EFFECT_DATE_TIME_FORMATTER = DateTimeFormatter.ofPattern("dd.MM.yyyy HH:mm:ss")
        private val FINANCIAL_EFFECT_TARGET_MONTH_FORMATTER = DateTimeFormatter.ofPattern("dd.MM.yyyy")

        fun buildFileName(now: LocalDateTime = LocalDateTime.now()): String =
            FILE_NAME_TEMPLATE.format(now.format(FILE_NAME_TIMESTAMP_FORMATTER))
    }

    /**
     * Формирует XLSX-реестр AI-инициатив с двумя листами.
     *
     * В выгрузку попадают только активные инициативы (disabled = false).
     * Первый лист содержит данные инициатив, включая бизнес-процессы,
     * бизнес-владельцев, IT-владельцев и статусы этапов.
     * Второй лист содержит активные финансовые эффекты выгружаемых инициатив.
     */
    @Transactional(readOnly = true)
    fun downloadRegistryExcel(): ResponseEntity<InputStreamResource> {
        val urlTemplate = buildInitiativeUrlTemplate(portalUrl)

        val agents = aiAgentRepository.findAllByDisabledIsFalse()
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

        val workbook = ExcelExportHelper.createWorkBook(listOf(SHEET_NAME, FINANCIAL_EFFECT_SHEET_NAME))

        ExcelExportHelper.writeSheetData(
            workbook, workbook.getSheetAt(0), initiativeRows, headerColumns()
        )

        ExcelExportHelper.writeSheetData(
            workbook, workbook.getSheetAt(1), financialEffectRows, getFinancialEffectHeaderColumns()
        )

        return with(ExcelExportHelper) {
            workbook.convertToFile(buildFileName(), XLSX_MEDIA_TYPE)
        }
    }

    /**
     * Загружает активные финансовые эффекты всех выгружаемых инициатив.
     * Используется один запрос независимо от количества инициатив.
     */
    private fun loadFinancialEffectRows(
        initiativeIds: Collection<Long>
    ): List<InitiativeFinancialEffectExcelExportModel> {
        if (initiativeIds.isEmpty()) return emptyList()

        return financialEffectRepository.findActiveByInitiativeIds(initiativeIds)
            .map { financialEffect -> buildFinancialEffectExcelExportModel(financialEffect) }
    }

    /**
     * Преобразует финансовый эффект в строку Excel.
     * Срок достижения выводится последним днём указанного месяца.
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
            targetMonth = financialEffect.targetMonth?.let {
                YearMonth.from(it).atEndOfMonth().format(FINANCIAL_EFFECT_TARGET_MONTH_FORMATTER)
            },
            approvalLinks = financialEffect.approvalLinks?.joinToString("; ") { it.asText() }.orEmpty(),
            createdAt = financialEffect.created?.format(FINANCIAL_EFFECT_DATE_TIME_FORMATTER)
        )
    }

    /**
     * Описание столбцов листа «Финансовый эффект».
     * План и факт записываются как числа, отсутствующий факт оставляет пустую ячейку.
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
     * Загружает контактные связи инициатив одним запросом.
     */
    private fun loadContactsByInitiativeId(
        initiativeIds: Collection<Long>
    ): Map<Long, List<InitiativeContactExportProjection>> {
        if (initiativeIds.isEmpty()) return emptyMap()

        return agentContactRepository.findContactsForRegistryExport(initiativeIds)
            .groupBy { it.initiativeId }
    }

    /**
     * Загружает коды бизнес-процессов (ProcessEntity.shortName) одним запросом.
     * Удаляет дубликаты и сортирует коды.
     */
    private fun loadBusinessProcessCodesByInitiativeId(
        initiativeIds: Collection<Long>
    ): Map<Long, List<String>> {
        if (initiativeIds.isEmpty()) return emptyMap()

        return aiAgentRepository.findBusinessProcessCodesForRegistryExport(initiativeIds)
            .groupBy { it.initiativeId }
            .mapValues { (_, processRows) ->
                processRows.mapNotNull { it.processCode?.trim()?.takeIf(String::isNotBlank) }
                    .distinct()
                    .sorted()
            }
    }

    /**
     * Загружает email пользователей АУЕ для контактов без contact.email.
     * Все уникальные userId запрашиваются одним bulk-вызовом.
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
                val email = user.email?.trim()?.takeIf { it.isNotBlank() } ?: return@mapNotNull null
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
     * Старые колонки финансового эффекта исключены.
     * Бизнес-процесс, бизнес-владелец и IT-владелец расположены
     * непосредственно после колонки «Поверхности».
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
            ExcelExportHelper.textColumn("Бизнес-процесс") { it.businessProcesses },
            ExcelExportHelper.textColumn("Бизнес-владелец") { it.businessOwner },
            ExcelExportHelper.textColumn("IT-владелец") { it.itOwner },
            ExcelExportHelper.textColumn("Архив") {
                when (it.isDisable) {
                    true -> "Да"
                    false -> "Нет"
                    null -> null
                }
            },
            ExcelExportHelper.hyperlinkColumn("Ссылка на инициативу в Пульт") { it.initiativeUrl },
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
