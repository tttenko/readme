```java


@Query("""
    SELECT DISTINCT j.agent.id AS agentId, j.jiraKey AS jiraKey
    FROM JiraIssueEntity j
    WHERE lower(j.project) = 'crossgoal'
      AND lower(j.type) = 'initiative'
      AND j.jiraKey IN :jiraKeys
      AND j.agent IS NOT NULL
""")
fun findMetaAiObjectsInitiativesByCrossgoalKeys(
    @Param("jiraKeys") jiraKeys: Set<String>
): List<CrossgoalAgentIdProjection>

@Repository
interface InitiativeAiObjectHistoryRepository : JpaRepository<InitiativeAiObjectHistoryEntity, Long>

@Repository
interface InitiativeMetaAiObjectRepository : JpaRepository<InitiativeMetaAiObjectEntity, Long> {
    @Query("""
        SELECT obj.ciSolution
        FROM InitiativeMetaAiObjectEntity obj
        WHERE obj.initiative.id = :initiativeId
          AND obj.isActive = true
          AND obj.ciSolution IS NOT NULL
    """)
    fun findActiveCiSolutionValuesByInitiativeId(@Param("initiativeId") initiativeId: Long): List<String>

    @Query("""
        SELECT obj
        FROM InitiativeMetaAiObjectEntity obj
        WHERE obj.initiative.id = :initiativeId
          AND obj.ciSolution IS NOT NULL
    """)
    fun findAllWithCiSolutionByInitiativeId(@Param("initiativeId") initiativeId: Long): List<InitiativeMetaAiObjectEntity>
}


import org.apache.poi.hssf.usermodel.HSSFWorkbook
import org.apache.poi.ss.usermodel.Cell
import org.apache.poi.ss.usermodel.CellType
import org.apache.poi.ss.usermodel.DataFormatter
import org.apache.poi.ss.usermodel.DateUtil
import org.apache.poi.ss.usermodel.Row
import org.apache.poi.ss.usermodel.Sheet
import org.apache.poi.ss.usermodel.Workbook
import org.springframework.stereotype.Component
import java.nio.file.Files
import java.nio.file.Path
import java.util.Locale

data class MetaAiObjectsExcelRow(
    val rowNumber: Int,
    val crossgoal: String,
    val ciSolution: String,
    val name: String?,
    val isAgent: Boolean?,
    val gigausage: String?,
    val metaLink: String?
)

data class MetaAiObjectsParsedFile(
    val rows: List<MetaAiObjectsExcelRow>,
    val errors: List<String>
)

class MetaAiObjectsInvalidFileException(message: String) : RuntimeException(message)

@Component
class MetaAiObjectsExcelReader {
    private enum class Column(val header: String) {
        CROSSGOAL("CROSSGOAL"),
        CI_SOLUTION("КЭ модуля"),
        NAME("Наименование"),
        IS_AGENT("Является агентом"),
        GIGAUSAGE("GigaUsage ID"),
        META_LINK("Ссылка на META")
    }

    fun validateStructure(source: Path, extension: String) {
        withSheet(source, extension) { sheet, formatter ->
            val iterator = sheet.iterator()
            if (!iterator.hasNext()) throw MetaAiObjectsInvalidFileException("В файле нет заголовков")
            parseHeader(iterator.next(), formatter)
            var hasData = false
            while (iterator.hasNext()) {
                val row = iterator.next()
                if (hasNonemptyCell(row, formatter)) {
                    hasData = true
                    break
                }
            }
            if (!hasData) throw MetaAiObjectsInvalidFileException("В файле отсутствуют строки данных")
            // Предпроверка не разбирает строки — их ошибки попадут в file_upload после 201.
        }
    }

    fun read(source: Path, extension: String): MetaAiObjectsParsedFile =
        withSheet(source, extension) { sheet, formatter ->
            val iterator = sheet.iterator()
            if (!iterator.hasNext()) throw MetaAiObjectsInvalidFileException("В файле нет заголовков")
            val columns = parseHeader(iterator.next(), formatter)
            val rows = mutableListOf<MetaAiObjectsExcelRow>()
            val errors = mutableListOf<String>()

            while (iterator.hasNext()) {
                val row = iterator.next()
                val number = row.rowNum + 1
                try {
                    val crossgoal = text(row, columns[Column.CROSSGOAL], formatter)
                    val ciSolution = text(row, columns[Column.CI_SOLUTION], formatter)
                    val name = text(row, columns[Column.NAME], formatter)
                    val agentRaw = text(row, columns[Column.IS_AGENT], formatter)
                    val gigausage = text(row, columns[Column.GIGAUSAGE], formatter)
                    val metaLink = text(row, columns[Column.META_LINK], formatter)
                    if (listOf(crossgoal, ciSolution, name, agentRaw, gigausage, metaLink).all { it == null }) {
                        continue
                    }
                    if (crossgoal == null) throw IllegalArgumentException("не заполнен CROSSGOAL")
                    if (ciSolution == null) throw IllegalArgumentException("не заполнен КЭ модуля")
                    if (crossgoal.length > 255) throw IllegalArgumentException("CROSSGOAL длиннее 255 символов")
                    if (name != null && name.length > 255) throw IllegalArgumentException("Наименование длиннее 255 символов")
                    if (gigausage != null && gigausage.length > 255) throw IllegalArgumentException("GigaUsage ID длиннее 255 символов")
                    if (metaLink != null && metaLink.length > 255) throw IllegalArgumentException("Ссылка на META длиннее 255 символов")
                    val isAgent = when (agentRaw?.lowercase()) {
                        null -> null
                        "true" -> true
                        "false" -> false
                        else -> throw IllegalArgumentException("Является агентом: допустимы true, false или пустое значение")
                    }
                    rows += MetaAiObjectsExcelRow(number, crossgoal, ciSolution, name, isAgent, gigausage, metaLink)
                } catch (e: Exception) {
                    errors += "Строка $number: ${e.message ?: "некорректные данные"}"
                }
            }
            MetaAiObjectsParsedFile(rows, errors)
        }

    private fun parseHeader(row: Row, formatter: DataFormatter): Map<Column, Int> {
        val result = mutableMapOf<Column, Int>()
        row.forEach { cell ->
            val label = text(cell, formatter)?.trim() ?: return@forEach
            val column = Column.entries.firstOrNull { it.header.equals(label, ignoreCase = true) }
                ?: return@forEach
            if (result.putIfAbsent(column, cell.columnIndex) != null) {
                throw MetaAiObjectsInvalidFileException("Дублируется колонка: ${column.header}")
            }
        }
        listOf(Column.CROSSGOAL, Column.CI_SOLUTION).forEach { column ->
            if (!result.containsKey(column)) {
                throw MetaAiObjectsInvalidFileException("Отсутствует обязательная колонка: ${column.header}")
            }
        }
        return result
    }

    private fun hasNonemptyCell(row: Row, formatter: DataFormatter): Boolean =
        row.any { cell -> text(cell, formatter) != null }

    private fun text(row: Row, columnIndex: Int?, formatter: DataFormatter): String? =
        columnIndex?.let { text(row.getCell(it), formatter) }

    private fun text(cell: Cell?, formatter: DataFormatter): String? {
        if (cell == null) return null
        val type = if (cell.cellType == CellType.FORMULA) cell.cachedFormulaResultType else cell.cellType
        val raw = when (type) {
            CellType.STRING -> cell.stringCellValue
            CellType.BOOLEAN -> cell.booleanCellValue.toString()
            CellType.NUMERIC -> {
                if (DateUtil.isCellDateFormatted(cell)) throw IllegalArgumentException("Дата вместо текстового значения")
                if (cell.cellType == CellType.FORMULA) {
                    val style = cell.cellStyle
                    formatter.formatRawCellContents(cell.numericCellValue, style.dataFormat.toInt(), style.dataFormatString ?: "General")
                } else {
                    formatter.formatCellValue(cell)
                }
            }
            CellType.BLANK -> ""
            CellType.ERROR -> throw IllegalArgumentException("Ошибка в ячейке Excel")
            else -> throw IllegalArgumentException("Неподдерживаемый тип ячейки")
        }
        return raw.trim().takeIf { it.isNotEmpty() }
    }

    private fun <T> withSheet(source: Path, extension: String, block: (Sheet, DataFormatter) -> T): T {
        Files.newInputStream(source).use { input ->
            val workbook: Workbook = if (extension == "xls") {
                HSSFWorkbook(input)
            } else {
                StreamingReader.builder().rowCacheSize(100).bufferSize(4096).open(input)
            }
            workbook.use {
                if (workbook.numberOfSheets == 0) throw MetaAiObjectsInvalidFileException("В Excel-файле нет листов")
                return block(workbook.getSheetAt(0), DataFormatter(Locale.ROOT))
            }
        }
    }
}


import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Propagation
import org.springframework.transaction.annotation.Transactional

@Service
class MetaAiObjectsImportApplyService(
    private val aiAgentRepository: AIAgentRepository,
    private val metaAiObjectRepository: InitiativeMetaAiObjectRepository,
    private val historyRepository: InitiativeAiObjectHistoryRepository
) {

    // Одна транзакция на инициативу: изменения объектов и история фиксируются вместе.
    // Независимые инициативы не откатывают друг друга при ошибке одной из них.
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    fun apply(initiativeId: Long, rows: List<MetaAiObjectsExcelRow>, fileId: Long, userId: Long): Int {
        val initiative = aiAgentRepository.findByIdForUpdate(initiativeId)
            ?: throw IllegalArgumentException("Инициатива $initiativeId больше не существует")

        // Последняя строка по ключу (initiative_id, ci_solution) определяет конечное состояние.
        // Это исключает лишние UPDATE/историю, если дубли в итоге вернули исходные значения.
        val finalRows = linkedMapOf<String, MetaAiObjectsExcelRow>()
        rows.forEach { finalRows[it.ciSolution] = it }

        val storedByCi = metaAiObjectRepository.findAllWithCiSolutionByInitiativeId(initiativeId)
            .associateBy { it.ciSolution }

        var changedCount = 0
        for (row in finalRows.values) {
            val stored = storedByCi[row.ciSolution]
            if (stored == null) {
                metaAiObjectRepository.save(
                    InitiativeMetaAiObjectEntity(
                        initiative = initiative,
                        crossgoal = row.crossgoal,
                        ciSolution = row.ciSolution,
                        name = row.name,
                        isAgent = row.isAgent,
                        gigausage = row.gigausage,
                        metaLink = row.metaLink,
                        source = InitiativeMetaAiObjectSource.META,
                        isActive = true,
                        createdBy = userId.toString()
                    )
                )
                changedCount++
            } else {
                val changed = stored.crossgoal != row.crossgoal ||
                    stored.name != row.name ||
                    stored.isAgent != row.isAgent ||
                    stored.gigausage != row.gigausage ||
                    stored.metaLink != row.metaLink ||
                    !stored.isActive
                if (changed) {
                    stored.crossgoal = row.crossgoal
                    stored.name = row.name
                    stored.isAgent = row.isAgent
                    stored.gigausage = row.gigausage
                    stored.metaLink = row.metaLink
                    stored.isActive = true
                    stored.updatedBy = userId.toString()
                    // stored — managed JPA entity; dirty checking выполнит UPDATE.
                    changedCount++
                }
            }
        }

        if (changedCount > 0) {
            historyRepository.save(
                InitiativeAiObjectHistoryEntity(
                    initiative = initiative,
                    fileUploadId = fileId,
                    changedObjectsCount = changedCount,
                    eventType = InitiativeAiObjectEventType.FILE_IMPORT,
                    createdBy = userId.toString(),
                    createdByName = null
                )
            )
        }
        return changedCount
    }
}


import org.springframework.stereotype.Service
import java.nio.file.Files
import java.nio.file.Path

@Service
class MetaAiObjectsImportWorker(
    private val reader: MetaAiObjectsExcelReader,
    private val jiraIssueRepository: JiraIssueRepository,
    private val applyService: MetaAiObjectsImportApplyService,
    private val fileUploadStatusService: FileUploadStatusService
) {
    private val log = org.slf4j.LoggerFactory.getLogger(javaClass)

    // Запускается выделенным executor — не через self-invocation @Async.
    fun process(source: Path, extension: String, fileId: Long, userId: Long) {
        try {
            val parsed = reader.read(source, extension)
            val errors = parsed.errors.toMutableList()

            val keys = parsed.rows.map { it.crossgoal }.toSet()
            val mappings = if (keys.isEmpty()) emptyList() else jiraIssueRepository.findMetaAiObjectsInitiativesByCrossgoalKeys(keys)
            val initiativesByCrossgoal = mappings.groupBy { it.jiraKey }
                .mapValues { (_, values) -> values.map { it.agentId }.toSet() }

            val rowsByInitiative = linkedMapOf<Long, MutableList<MetaAiObjectsExcelRow>>()
            parsed.rows.forEach { row ->
                val initiativeIds = initiativesByCrossgoal[row.crossgoal].orEmpty()
                when {
                    initiativeIds.isEmpty() -> errors += "Строка ${row.rowNumber}: инициатива CROSSGOAL=${row.crossgoal} не найдена"
                    initiativeIds.size != 1 -> errors += "Строка ${row.rowNumber}: CROSSGOAL=${row.crossgoal} соответствует нескольким инициативам"
                    else -> rowsByInitiative.getOrPut(initiativeIds.single()) { mutableListOf() }.add(row)
                }
            }

            var changedTotal = 0
            for ((initiativeId, initiativeRows) in rowsByInitiative) {
                try {
                    changedTotal += applyService.apply(initiativeId, initiativeRows, fileId, userId)
                } catch (e: Exception) {
                    log.error("Meta AI objects import failed for initiativeId={}, fileId={}", initiativeId, fileId, e)
                    errors += "Инициатива $initiativeId: ${e.message ?: "ошибка сохранения"}"
                }
            }

            if (errors.isEmpty()) {
                fileUploadStatusService.updateStatus(fileId, "success", null, userId)
                log.info("Meta AI objects import completed: fileId={}, changedObjects={}", fileId, changedTotal)
            } else {
                val description = "Ошибок: ${errors.size}. " + errors.take(20).joinToString("; ")
                fileUploadStatusService.updateStatus(fileId, "error", description, userId)
                log.warn("Meta AI objects import finished with errors: fileId={}, changedObjects={}, errors={}", fileId, changedTotal, errors.size)
            }
        } catch (e: Exception) {
            log.error("Meta AI objects import failed: fileId={}", fileId, e)
            try {
                fileUploadStatusService.updateStatus(fileId, "error", e.message, userId)
            } catch (statusError: Exception) {
                log.error("Could not update file_upload status: fileId={}", fileId, statusError)
            }
        } finally {
            try {
                Files.deleteIfExists(source)
            } catch (e: Exception) {
                log.warn("Could not delete temporary Excel: {}", source, e)
            }
        }
    }
}


import org.springframework.http.HttpStatus
import org.springframework.http.MediaType
import org.springframework.http.ResponseEntity
import org.springframework.security.access.prepost.PreAuthorize
import org.springframework.web.bind.annotation.PostMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RequestParam
import org.springframework.web.bind.annotation.RestController
import org.springframework.web.multipart.MultipartFile

@RestController
@RequestMapping("/api/ai/v1/admin")
class MetaAiObjectsUploadController(
    private val metaAiObjectsUploadService: MetaAiObjectsUploadService
) {

    @PostMapping("/meta-ai-objects-upload", consumes = [MediaType.MULTIPART_FORM_DATA_VALUE])
    @PreAuthorize("hasAnyAuthority('TRANSFORMATION_OFFICE')")
    fun upload(@RequestParam("file") file: MultipartFile): ResponseEntity<MetaAiObjectsUploadResponse> {
        val fileId = metaAiObjectsUploadService.upload(file)
        return ResponseEntity.status(HttpStatus.CREATED).body(MetaAiObjectsUploadResponse(fileId))
    }
}

data class MetaAiObjectsUploadResponse(val fileId: Long)


import org.apache.poi.poifs.filesystem.FileMagic
import org.springframework.http.HttpStatus
import org.springframework.stereotype.Service
import org.springframework.web.multipart.MultipartFile
import org.springframework.web.server.ResponseStatusException
import java.nio.file.Files
import java.nio.file.Path
import java.util.concurrent.ArrayBlockingQueue
import java.util.concurrent.RejectedExecutionException
import java.util.concurrent.ThreadPoolExecutor
import java.util.concurrent.TimeUnit
import jakarta.annotation.PreDestroy

@Service
class MetaAiObjectsUploadService(
    private val fileUploadStatusService: FileUploadStatusService,
    private val userInfoProvider: UserInfoProvider,
    private val excelReader: MetaAiObjectsExcelReader,
    private val importWorker: MetaAiObjectsImportWorker
) {
    private companion object {
        const val MAX_FILE_SIZE_BYTES = 20L * 1024 * 1024
        const val MAX_FILE_NAME_LENGTH = 255
    }

    // Отдельная очередь, чтобы не затронуть импорт общего реестра AI-агентов.
    // При переполнении очереди не регистрируем задачу как успешно принятую.
    private val executor = ThreadPoolExecutor(
        1,
        1,
        0L,
        TimeUnit.MILLISECONDS,
        ArrayBlockingQueue(10),
        { task -> Thread(task, "meta-ai-objects-import-worker").apply { isDaemon = false } },
        ThreadPoolExecutor.AbortPolicy()
    )

    fun upload(file: MultipartFile): Long {
        val fileName = file.originalFilename.orEmpty().replace('\\', '/').substringAfterLast('/')
        val extension = fileName.substringAfterLast('.', "").lowercase()
        if (extension !in setOf("xls", "xlsx")) {
            throw ResponseStatusException(HttpStatus.BAD_REQUEST, "Допустимы только файлы .xls и .xlsx")
        }
        if (fileName.length > MAX_FILE_NAME_LENGTH) {
            throw ResponseStatusException(HttpStatus.BAD_REQUEST, "Имя файла длиннее 255 символов")
        }
        if (file.isEmpty || file.size == 0L || file.size > MAX_FILE_SIZE_BYTES) {
            throw ResponseStatusException(HttpStatus.BAD_REQUEST, "Файл пустой или превышает 20 МБ")
        }

        val source = Files.createTempFile("meta-ai-objects-upload-", ".$extension")
        var submitted = false
        try {
            file.inputStream.use { input ->
                Files.newOutputStream(source).use { output -> input.copyTo(output) }
            }
            validateMagic(source, extension)

            // До создания file_upload проверяем заголовок и наличие хотя бы одной строки данных.
            try {
                excelReader.validateStructure(source, extension)
            } catch (e: MetaAiObjectsInvalidFileException) {
                throw ResponseStatusException(HttpStatus.BAD_REQUEST, e.message, e)
            } catch (e: Exception) {
                throw ResponseStatusException(HttpStatus.BAD_REQUEST, "Повреждённый Excel-файл", e)
            }

            val userId = userInfoProvider.currentUser().id
            val fileId = fileUploadStatusService.init(file, userId).id
            try {
                executor.execute { importWorker.process(source, extension, fileId, userId) }
                submitted = true
                return fileId
            } catch (rejected: RejectedExecutionException) {
                // Запись уже зарегистрирована отдельной транзакцией: нужно явно пометить ошибку.
                fileUploadStatusService.updateStatus(fileId, "error", "Очередь импорта переполнена", userId)
                throw ResponseStatusException(HttpStatus.INTERNAL_SERVER_ERROR, "Не удалось запустить обработку файла", rejected)
            }
        } finally {
            if (!submitted) Files.deleteIfExists(source)
        }
    }

    private fun validateMagic(source: Path, extension: String) {
        val magic = try {
            Files.newInputStream(source).use { input ->
                FileMagic.valueOf(FileMagic.prepareToCheckMagic(input))
            }
        } catch (e: Exception) {
            throw ResponseStatusException(HttpStatus.BAD_REQUEST, "Повреждённый Excel-файл", e)
        }
        val matches = (extension == "xlsx" && magic == FileMagic.OOXML) ||
            (extension == "xls" && magic == FileMagic.OLE2)
        if (!matches) {
            throw ResponseStatusException(HttpStatus.BAD_REQUEST, "Расширение файла не соответствует содержимому")
        }
    }

    @PreDestroy
    fun shutdown() {
        executor.shutdown()
    }
}


```
