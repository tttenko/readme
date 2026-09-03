```java
/**
 * Описание Excel-колонки.
 *
 * @param label заголовок колонки.
 * @param columnFun функция заполнения ячейки значением сущности.
 * В функцию передаются сущность, workbook и текущая ячейка.
 * @param columnStyleFun опциональная функция создания стиля ячейки.
 */
data class ExcelColumnDescription<T>(
    val label: String,
    val columnFun: (param: Triple<T, XSSFWorkbook, Cell>) -> Unit,
    val columnStyleFun: Optional<(XSSFWorkbook) -> XSSFCellStyle> = Optional.empty(),
)

/**
 * Общий helper для формирования Excel-файлов.
 *
 * Отвечает за:
 * - создание workbook и листов;
 * - формирование заголовков таблицы;
 * - заполнение строк данными;
 * - создание типовых текстовых, числовых и hyperlink-колонок;
 * - преобразование workbook в HTTP-response для скачивания файла.
 */
object ExcelExportHelper {

    /**
     * Создаёт новый Excel workbook с указанными листами.
     *
     * @param sheets список названий листов.
     */
    fun createWorkBook(sheets: List<String>): XSSFWorkbook {
        val workbook = XSSFWorkbook()
        sheets.forEach { workbook.createSheet(it) }
        return workbook
    }

    /**
     * Формирует строку заголовков таблицы.
     *
     * @return количество созданных строк.
     */
    private fun <T> renderTableHeading(
        sheet: XSSFSheet,
        columnFuns: List<ExcelColumnDescription<T>>,
        startRowNum: Int = 0,
        headerColumnStyle: XSSFCellStyle?,
    ): Int {
        val row = sheet.createRow(startRowNum)

        columnFuns.forEachIndexed { index, columnDescription ->
            val cell = row.createCell(index, CellType.STRING)
            cell.setCellValue(columnDescription.label)
            headerColumnStyle?.let { cell.cellStyle = it }
        }

        return 1
    }

    /**
     * Формирует одну строку таблицы для переданной сущности.
     *
     * Значение каждой ячейки определяется соответствующим [ExcelColumnDescription].
     *
     * @return количество созданных строк.
     */
    private fun <T> renderEntityRows(
        workBook: XSSFWorkbook,
        sheet: XSSFSheet,
        columnFuns: List<ExcelColumnDescription<T>>,
        columnStyles: List<Optional<XSSFCellStyle>>,
        entity: T,
        rowNum: Int,
    ): Int {
        val row = sheet.createRow(rowNum)

        columnFuns.forEachIndexed { index, columnDescription ->
            val cell = row.createCell(index)
            columnDescription.columnFun(Triple(entity, workBook, cell))
            columnStyles[index].ifPresent { cell.cellStyle = it }
        }

        return 1
    }

    /**
     * Записывает в лист заголовок и все строки переданного набора данных.
     *
     * @param workBook workbook, которому принадлежит лист.
     * @param sheet лист для записи данных.
     * @param data данные для формирования строк.
     * @param columnDescriptions описание колонок и правил заполнения ячеек.
     * @param headerColumnStyle опциональный стиль строки заголовков.
     */
    fun <T> writeSheetData(
        workBook: XSSFWorkbook,
        sheet: XSSFSheet,
        data: List<T>,
        columnDescriptions: List<ExcelColumnDescription<T>>,
        headerColumnStyle: XSSFCellStyle? = null,
    ) {
        val headingRowsCount = renderTableHeading(sheet, columnDescriptions, headerColumnStyle = headerColumnStyle)
        val columnStyles = columnDescriptions.map { columnDescription -> columnDescription.columnStyleFun.map { it(workBook) } }

        data.fold(headingRowsCount) { rowNum, entity ->
            rowNum + renderEntityRows(workBook, sheet, columnDescriptions, columnStyles, entity, rowNum)
        }
    }

    /**
     * Создаёт текстовую Excel-колонку.
     *
     * Если [valueProvider] возвращает null, ячейка остаётся пустой.
     */
    fun <T> textColumn(label: String, valueProvider: (T) -> String?): ExcelColumnDescription<T> =
        ExcelColumnDescription(label) { (entity, _, cell) ->
            valueProvider(entity)?.let { cell.setCellValue(it) }
        }

    /**
     * Создаёт числовую Excel-колонку.
     *
     * Числовое значение записывается в ячейку как Double.
     * Если [valueProvider] возвращает null, ячейка остаётся пустой.
     */
    fun <T> numberColumn(label: String, valueProvider: (T) -> Number?): ExcelColumnDescription<T> =
        ExcelColumnDescription(label) { (entity, _, cell) ->
            valueProvider(entity)?.let { cell.setCellValue(it.toDouble()) }
        }

    /**
     * Создаёт Excel-колонку с активной URL-гиперссылкой.
     *
     * URL одновременно используется как отображаемое значение ячейки
     * и как адрес Excel hyperlink.
     */
    fun <T> hyperlinkColumn(label: String, valueProvider: (T) -> String?): ExcelColumnDescription<T> =
        ExcelColumnDescription(label) { (entity, workbook, cell) ->
            valueProvider(entity)?.takeIf { it.isNotBlank() }?.let { url ->
                cell.setCellValue(url)
                cell.hyperlink = workbook.creationHelper.createHyperlink(HyperlinkType.URL).apply { address = url }
            }
        }

    /**
     * Преобразует workbook в HTTP-response со скачиваемым файлом.
     *
     * Workbook записывается в память и закрывается после формирования массива байт.
     * Заголовок Content-Disposition содержит UTF-8 имя файла.
     *
     * @param fileName имя скачиваемого файла.
     * @param mediaType Content-Type ответа. По умолчанию используется
     * [MediaType.APPLICATION_OCTET_STREAM] для обратной совместимости существующих экспортов.
     */
    fun XSSFWorkbook.convertToFile(
        fileName: String,
        mediaType: MediaType = MediaType.APPLICATION_OCTET_STREAM,
    ): ResponseEntity<InputStreamResource> {
        val baos = ByteArrayOutputStream()
        this.use { it.write(baos) }

        val data = baos.toByteArray()
        val contentDisposition = ContentDisposition.attachment().filename(fileName, StandardCharsets.UTF_8).build()

        val headers = HttpHeaders().apply {
            this.contentDisposition = contentDisposition
            this.contentLength = data.size.toLong()
            this.contentType = mediaType
            this.accessControlExposeHeaders = listOf(HttpHeaders.CONTENT_DISPOSITION)
        }

        return ResponseEntity.ok()
            .headers(headers)
            .body(InputStreamResource(ByteArrayInputStream(data)))
    }
}

/**
 * Сервис формирования полного XLSX-реестра AI-инициатив.
 *
 * Используется двумя endpoint:
 * - GET /api/v1/ai-agent/initiatives/export для TRANSFORMATION_OFFICE;
 * - GET /api/v1/admin/ai-agent-download для CMS_ADMIN.
 *
 * Выгрузка содержит все инициативы, включая архивные, и формирует 28 колонок:
 * 17 базовых колонок инициативы и 11 дополнительных колонок реестра.
 *
 * Для получения SLA используется один bulk-запрос по всем выгружаемым инициативам.
 */
@Service
class InitiativeRegistryExportService(
    private val mapper: InitiativeRegistryMapper,
    private val terBankService: TerBankService,
    private val aiAgentRepository: AIAgentRepository,
    private val statusRepository: StatusRepository,
    private val agentStatusSlaRepository: AgentStatusSlaRepository,
    private val pultProperties: PultProperties,
) {

    @Value("\${scheduled.jira-sync.delta.url.prefix}")
    private var deltaPrefix: String = ""

    @Value("\${scheduled.jira-sync.sigma.url.prefix}")
    private var sigmaPrefix: String = ""

    companion object {
        /** MIME-type XLSX-файла. */
        const val XLSX_MEDIA_TYPE_VALUE = "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"

        /** MediaType XLSX-файла для HTTP-response. */
        val XLSX_MEDIA_TYPE: MediaType = MediaType.parseMediaType(XLSX_MEDIA_TYPE_VALUE)

        /** Имя файла полного реестра AI-инициатив. */
        const val FILE_NAME = "Портфель AI-инициатив.xlsx"

        /** Название листа внутри сформированного Excel-файла. */
        private const val SHEET_NAME = "Портфель AI-инициатив"
    }

    /**
     * Формирует полный XLSX-реестр AI-инициатив.
     *
     * В выгрузку включаются все инициативы, включая disabled.
     * Фильтры, поиск и пагинация каталога не применяются.
     *
     * Алгоритм:
     * 1. Проверить конфигурацию шаблона ссылки Пульта.
     * 2. Получить полный список инициатив.
     * 3. Получить справочник тербанков.
     * 4. Получить ordering активных статусов.
     * 5. Одним запросом получить SLA всех инициатив.
     * 6. Сформировать модели строк через [InitiativeRegistryMapper].
     * 7. Сформировать Excel и вернуть его как XLSX-файл.
     */
    @Transactional(readOnly = true)
    fun downloadRegistryExcel(): ResponseEntity<InputStreamResource> {
        val urlTemplate = pultProperties.initiativeUrlTemplate
        validateUrlTemplate(urlTemplate)

        val agents = aiAgentRepository.findAll()
        val terBanks = terBankService.list()
        val statusOrderingByCode = loadStatusOrderingByCode()
        val slaByInitiativeAndStatusCode = loadSlaByInitiativeAndStatusCode(agents.map { it.id })

        val data = agents.map { agent ->
            mapper.toInitiativeRegistryExcelExportModel(
                entity = agent,
                terBanks = terBanks,
                deltaPrefix = deltaPrefix,
                sigmaPrefix = sigmaPrefix,
                statusOrderingByCode = statusOrderingByCode,
                slaByStatusCode = slaByInitiativeAndStatusCode[agent.id].orEmpty(),
                urlTemplate = urlTemplate,
            )
        }

        val workbook = ExcelExportHelper.createWorkBook(listOf(SHEET_NAME))
        ExcelExportHelper.writeSheetData(workbook, workbook.getSheetAt(0), data, headerColumns())

        return with(ExcelExportHelper) {
            workbook.convertToFile(FILE_NAME, XLSX_MEDIA_TYPE)
        }
    }

    /**
     * Проверяет конфигурацию ссылки на инициативу в Пульте.
     *
     * Шаблон должен быть непустым и содержать placeholder `{id}`,
     * который при формировании строки заменяется внутренним ID инициативы.
     *
     * @throws AiInternalServerException если property отсутствует или имеет некорректный формат.
     */
    private fun validateUrlTemplate(urlTemplate: String) {
        if (urlTemplate.isBlank() || !urlTemplate.contains("{id}")) {
            throw AiInternalServerException(
                message = "Property 'prm.pult.initiative-url-template' must contain '{id}'"
            )
        }
    }

    /**
     * Загружает ordering активных статусов и индексирует их по code.
     *
     * Используется для вычисления жизненного статуса этапов
     * «Завершён», «В работе» и «План».
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
     * Результат группируется сначала по ID инициативы,
     * затем по code статуса этапа:
     *
     * initiativeId -> statusCode -> AgentStatusSlaEntity.
     *
     * Такой подход исключает выполнение отдельного SLA-запроса
     * для каждой инициативы.
     */
    private fun loadSlaByInitiativeAndStatusCode(
        initiativeIds: Collection<Long>,
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
     * Описание 28 колонок файла «Портфель AI-инициатив.xlsx».
     *
     * Первые 17 колонок соответствуют существующей выгрузке AI-инициатив.
     * Далее добавляются:
     * - признак архива;
     * - ссылка на инициативу в Пульт;
     * - email контактов;
     * - статус и дедлайн этапов Концепция, PoC, MVP и Целевое решение.
     *
     * Колонка «Ссылка на инициативу в Пульт» записывается как активная Excel-гиперссылка.
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
            ExcelExportHelper.numberColumn("Фин. эффект, руб.") { it.effectRevenue },
            ExcelExportHelper.numberColumn("Фин. эффект, ПШЕ") { it.effectOptimization },
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
        )
}


```
