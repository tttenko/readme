```java

meta.ai.objects.upload.duplicate.column=Дублируется колонка {0}
meta.ai.objects.upload.required.value=Не заполнено обязательное поле {0}
meta.ai.objects.upload.max.length=Значение поля {0} превышает {1} символов
meta.ai.objects.upload.invalid.agent.type=Поле Является агентом должно содержать true, false или быть пустым
meta.ai.objects.upload.invalid.cell=Некорректное значение ячейки в колонке {0}
meta.ai.objects.upload.row.error=Строка {0}: {1}
meta.ai.objects.upload.file.name.too.long=Имя файла превышает {0} символов

const val META_AI_OBJECTS_UPLOAD_DUPLICATE_COLUMN = "meta.ai.objects.upload.duplicate.column"
const val META_AI_OBJECTS_UPLOAD_REQUIRED_VALUE = "meta.ai.objects.upload.required.value"
const val META_AI_OBJECTS_UPLOAD_MAX_LENGTH = "meta.ai.objects.upload.max.length"
const val META_AI_OBJECTS_UPLOAD_INVALID_AGENT_TYPE = "meta.ai.objects.upload.invalid.agent.type"
const val META_AI_OBJECTS_UPLOAD_INVALID_CELL = "meta.ai.objects.upload.invalid.cell"
const val META_AI_OBJECTS_UPLOAD_ROW_ERROR = "meta.ai.objects.upload.row.error"
const val META_AI_OBJECTS_UPLOAD_FILE_NAME_TOO_LONG = "meta.ai.objects.upload.file.name.too.long"


/** Ожидаемые названия колонок нового шаблона Excel. */
internal enum class MetaAiObjectsExcelColumn(val header: String) {
    CROSSGOAL("CROSSGOAL"),
    CI_SOLUTION("КЭ модуля"),
    NAME("Наименование"),
    IS_AGENT("Является агентом"),
    GIGAUSAGE("GigaUsage ID"),
    META_LINK("Ссылка на META")
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
import java.text.MessageFormat
import java.util.Locale

@Component
class MetaAiObjectsExcelReader(
    private val messageProvider: MessageProvider
) {
    private companion object {
        const val MAX_TEXT_LENGTH = 255
        val REQUIRED_COLUMNS = setOf(
            MetaAiObjectsExcelColumn.CROSSGOAL,
            MetaAiObjectsExcelColumn.CI_SOLUTION
        )
    }

    /** Структуру проверяем синхронно, до создания file_upload и ответа 201. */
    fun validateStructure(source: Path, extension: String) {
        withFirstSheet(source, extension) { sheet, formatter ->
            val rows = sheet.iterator()
            if (!rows.hasNext()) invalidFile(Metadata.ErrorMessages.AI_UPLOAD_EMPTY_FILE)

            readHeader(rows.next(), formatter)

            // На этом этапе не проверяем значения строк: ошибки строк относятся
            // к фоновой обработке и попадают в file_upload.error_description.
            val hasDataRow = rows.asSequence().any(::containsData)
            if (!hasDataRow) invalidFile(Metadata.ErrorMessages.AI_UPLOAD_EMPTY_FILE)
        }
    }

    fun read(source: Path, extension: String): MetaAiObjectsParsedFile =
        withFirstSheet(source, extension) { sheet, formatter ->
            val rows = sheet.iterator()
            if (!rows.hasNext()) invalidFile(Metadata.ErrorMessages.AI_UPLOAD_EMPTY_FILE)

            val indices = readHeader(rows.next(), formatter)
            val validRows = mutableListOf<MetaAiObjectsExcelRow>()
            val errors = mutableListOf<String>()

            while (rows.hasNext()) {
                val row = rows.next()
                val rowNumber = row.rowNum + 1
                try {
                    readDataRow(row, indices, formatter)?.let(validRows::add)
                } catch (failure: Exception) {
                    val reason = failure.message ?: messageProvider[Metadata.ErrorMessages.IncorrectDataFormat]
                    errors += format(Metadata.ErrorMessages.META_AI_OBJECTS_UPLOAD_ROW_ERROR, rowNumber, reason)
                }
            }

            MetaAiObjectsParsedFile(validRows, errors)
        }

    private fun readHeader(row: Row, formatter: DataFormatter): Map<MetaAiObjectsExcelColumn, Int> {
        val indices = mutableMapOf<MetaAiObjectsExcelColumn, Int>()

        row.forEach { cell ->
            val header = readCellText(cell, formatter, "Заголовок") ?: return@forEach
            val column = MetaAiObjectsExcelColumn.entries.firstOrNull {
                it.header.equals(header, ignoreCase = true)
            } ?: return@forEach

            if (indices.putIfAbsent(column, cell.columnIndex) != null) {
                invalidFile(Metadata.ErrorMessages.META_AI_OBJECTS_UPLOAD_DUPLICATE_COLUMN, column.header)
            }
        }

        REQUIRED_COLUMNS.forEach { column ->
            if (column !in indices) {
                invalidFile(Metadata.ErrorMessages.REQUIRED_COLUMN_NOT_FOUND, column.header)
            }
        }
        return indices
    }

    private fun readDataRow(
        row: Row,
        indices: Map<MetaAiObjectsExcelColumn, Int>,
        formatter: DataFormatter
    ): MetaAiObjectsExcelRow? {
        val crossgoal = readField(row, MetaAiObjectsExcelColumn.CROSSGOAL, indices, formatter)
        val ciSolution = readField(row, MetaAiObjectsExcelColumn.CI_SOLUTION, indices, formatter)
        val name = readField(row, MetaAiObjectsExcelColumn.NAME, indices, formatter)
        val isAgentValue = readField(row, MetaAiObjectsExcelColumn.IS_AGENT, indices, formatter)
        val gigausage = readField(row, MetaAiObjectsExcelColumn.GIGAUSAGE, indices, formatter)
        val metaLink = readField(row, MetaAiObjectsExcelColumn.META_LINK, indices, formatter)

        if (listOf(crossgoal, ciSolution, name, isAgentValue, gigausage, metaLink).all { it == null }) {
            return null
        }

        if (crossgoal == null) {
            invalidFile(Metadata.ErrorMessages.META_AI_OBJECTS_UPLOAD_REQUIRED_VALUE, MetaAiObjectsExcelColumn.CROSSGOAL.header)
        }
        if (ciSolution == null) {
            invalidFile(Metadata.ErrorMessages.META_AI_OBJECTS_UPLOAD_REQUIRED_VALUE, MetaAiObjectsExcelColumn.CI_SOLUTION.header)
        }

        checkLength(MetaAiObjectsExcelColumn.CROSSGOAL, crossgoal)
        checkLength(MetaAiObjectsExcelColumn.NAME, name)
        checkLength(MetaAiObjectsExcelColumn.GIGAUSAGE, gigausage)
        checkLength(MetaAiObjectsExcelColumn.META_LINK, metaLink)

        val isAgent = when (isAgentValue?.lowercase(Locale.ROOT)) {
            null -> null
            "true" -> true
            "false" -> false
            else -> invalidFile(Metadata.ErrorMessages.META_AI_OBJECTS_UPLOAD_INVALID_AGENT_TYPE)
        }

        return MetaAiObjectsExcelRow(
            rowNumber = row.rowNum + 1,
            crossgoal = crossgoal,
            ciSolution = ciSolution,
            name = name,
            isAgent = isAgent,
            gigausage = gigausage,
            metaLink = metaLink
        )
    }

    private fun readField(
        row: Row,
        column: MetaAiObjectsExcelColumn,
        indices: Map<MetaAiObjectsExcelColumn, Int>,
        formatter: DataFormatter
    ): String? {
        val index = indices[column] ?: return null
        return readCellText(row.getCell(index), formatter, column.header)
    }

    private fun readCellText(cell: Cell?, formatter: DataFormatter, fieldName: String): String? {
        if (cell == null) return null

        val value = try {
            val effectiveType = if (cell.cellType == CellType.FORMULA) cell.cachedFormulaResultType else cell.cellType
            when (effectiveType) {
                CellType.STRING -> cell.stringCellValue
                CellType.BOOLEAN -> cell.booleanCellValue.toString()
                CellType.NUMERIC -> {
                    if (DateUtil.isCellDateFormatted(cell)) {
                        invalidFile(Metadata.ErrorMessages.META_AI_OBJECTS_UPLOAD_INVALID_CELL, fieldName)
                    }
                    if (cell.cellType == CellType.FORMULA) {
                        val style = cell.cellStyle
                        formatter.formatRawCellContents(
                            cell.numericCellValue,
                            style.dataFormat.toInt(),
                            style.dataFormatString ?: "General"
                        )
                    } else {
                        formatter.formatCellValue(cell)
                    }
                }
                CellType.BLANK -> ""
                else -> invalidFile(Metadata.ErrorMessages.META_AI_OBJECTS_UPLOAD_INVALID_CELL, fieldName)
            }
        } catch (known: AiResponseException) {
            throw known
        } catch (failure: Exception) {
            invalidFile(Metadata.ErrorMessages.META_AI_OBJECTS_UPLOAD_INVALID_CELL, fieldName)
        }

        return value.trim().takeIf(String::isNotEmpty)
    }

    private fun checkLength(column: MetaAiObjectsExcelColumn, value: String?) {
        if (value != null && value.length > MAX_TEXT_LENGTH) {
            invalidFile(Metadata.ErrorMessages.META_AI_OBJECTS_UPLOAD_MAX_LENGTH, column.header, MAX_TEXT_LENGTH)
        }
    }

    private fun containsData(row: Row): Boolean = row.any { cell ->
        when (cell.cellType) {
            CellType.BLANK -> false
            CellType.STRING -> cell.stringCellValue.isNotBlank()
            else -> true // Ошибка содержимого строки фиксируется после 201, не здесь.
        }
    }

    private fun <T> withFirstSheet(
        source: Path,
        extension: String,
        block: (Sheet, DataFormatter) -> T
    ): T {
        try {
            Files.newInputStream(source).use { input ->
                val workbook: Workbook = when (extension) {
                    "xls" -> HSSFWorkbook(input)
                    "xlsx" -> StreamingReader.builder().rowCacheSize(100).bufferSize(4096).open(input)
                    else -> invalidFile(Metadata.ErrorMessages.FileNotValid)
                }
                workbook.use {
                    if (workbook.numberOfSheets == 0) invalidFile(Metadata.ErrorMessages.AI_UPLOAD_EMPTY_FILE)
                    return block(workbook.getSheetAt(0), DataFormatter(Locale.ROOT))
                }
            }
        } catch (known: AiResponseException) {
            throw known
        } catch (failure: Exception) {
            invalidFile(Metadata.ErrorMessages.AI_UPLOAD_CORRUPTED_FILE)
        }
    }

    private fun format(errorCode: String, vararg args: Any): String =
        MessageFormat.format(messageProvider[errorCode], *args)

    private fun invalidFile(errorCode: String, vararg args: Any): Nothing =
        throw AiBadRequestException(errorCode = errorCode, message = format(errorCode, *args))
}

/** Одна валидная строка XLS/XLSX после нормализации значений. */
data class MetaAiObjectsExcelRow(
    val rowNumber: Int,
    val crossgoal: String,
    val ciSolution: String,
    val name: String?,
    val isAgent: Boolean?,
    val gigausage: String?,
    val metaLink: String?
)


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

/** Результат фонового разбора: валидные строки и ошибки отдельных строк. */
data class MetaAiObjectsParsedFile(
    val rows: List<MetaAiObjectsExcelRow>,
    val errors: List<String>
)


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

import jakarta.annotation.PreDestroy
import org.apache.poi.poifs.filesystem.FileMagic
import org.slf4j.LoggerFactory
import org.springframework.http.HttpStatus
import org.springframework.stereotype.Service
import org.springframework.web.multipart.MultipartFile
import java.nio.file.Files
import java.nio.file.Path
import java.text.MessageFormat
import java.util.Locale
import java.util.concurrent.ArrayBlockingQueue
import java.util.concurrent.RejectedExecutionException
import java.util.concurrent.ThreadPoolExecutor
import java.util.concurrent.TimeUnit

@Service
class MetaAiObjectsUploadService(
    private val fileUploadStatusService: FileUploadStatusService,
    private val userInfoProvider: UserInfoProvider,
    private val excelReader: MetaAiObjectsExcelReader,
    private val importWorker: MetaAiObjectsImportWorker,
    private val messageProvider: MessageProvider
) {
    private companion object {
        const val MAX_FILE_SIZE_BYTES = 20L * 1024 * 1024
        const val MAX_FILE_NAME_LENGTH = 255
        val ALLOWED_EXTENSIONS = setOf("xls", "xlsx")
    }

    private val log = LoggerFactory.getLogger(javaClass)

    // Независимая очередь — импорт общего реестра AI-агентов не затрагиваем.
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
        val extension = fileName.substringAfterLast('.', "").lowercase(Locale.ROOT)

        if (extension !in ALLOWED_EXTENSIONS) badRequest(Metadata.ErrorMessages.FileNotValid)
        if (fileName.length > MAX_FILE_NAME_LENGTH) {
            badRequest(Metadata.ErrorMessages.META_AI_OBJECTS_UPLOAD_FILE_NAME_TOO_LONG, MAX_FILE_NAME_LENGTH)
        }
        if (file.isEmpty || file.size == 0L) badRequest(Metadata.ErrorMessages.AI_UPLOAD_EMPTY_FILE)
        if (file.size > MAX_FILE_SIZE_BYTES) badRequest(Metadata.ErrorMessages.AI_UPLOAD_MAX_FILE_SIZE)

        val source = Files.createTempFile("meta-ai-objects-upload-", ".$extension")
        var submitted = false
        try {
            file.inputStream.use { input ->
                Files.newOutputStream(source).use { output -> input.copyTo(output) }
            }

            validateMagic(source, extension)
            excelReader.validateStructure(source, extension)

            val userId = userInfoProvider.currentUser().id
            val fileId = fileUploadStatusService.init(file, userId).id

            try {
                executor.execute { importWorker.process(source, extension, fileId, userId) }
                submitted = true
                return fileId
            } catch (rejected: RejectedExecutionException) {
                runCatching {
                    fileUploadStatusService.updateStatus(fileId, "error", "Очередь загрузок переполнена", userId)
                }.onFailure { log.error("Could not mark rejected upload as ERROR: fileId={}", fileId, it) }

                throw AiResponseException(
                    status = HttpStatus.INTERNAL_SERVER_ERROR,
                    errorCode = Metadata.ErrorMessages.InternalServerError,
                    message = messageProvider[Metadata.ErrorMessages.InternalServerError]
                )
            }
        } finally {
            if (!submitted) {
                runCatching { Files.deleteIfExists(source) }
                    .onFailure { log.warn("Cannot delete temp upload file: {}", source, it) }
            }
        }
    }

    private fun validateMagic(source: Path, extension: String) {
        val magic = try {
            Files.newInputStream(source).use { input ->
                FileMagic.valueOf(FileMagic.prepareToCheckMagic(input))
            }
        } catch (failure: Exception) {
            badRequest(Metadata.ErrorMessages.AI_UPLOAD_CORRUPTED_FILE)
        }

        val matches = when (extension) {
            "xls" -> magic == FileMagic.OLE2
            "xlsx" -> magic == FileMagic.OOXML
            else -> false
        }
        if (!matches) badRequest(Metadata.ErrorMessages.FileNotValid)
    }

    private fun badRequest(errorCode: String, vararg args: Any): Nothing =
        throw AiBadRequestException(
            errorCode = errorCode,
            message = MessageFormat.format(messageProvider[errorCode], *args)
        )

    @PreDestroy
    fun shutdown() {
        executor.shutdown()
    }
}



```
