```java

import org.slf4j.LoggerFactory
import org.springframework.context.annotation.Bean
import org.springframework.scheduling.annotation.Async
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor
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

    private val log = LoggerFactory.getLogger(javaClass)

    @Bean("metaAiObjectsImportExecutor")
    fun metaAiObjectsImportExecutor() = ThreadPoolTaskExecutor().apply {
        corePoolSize = 1
        maxPoolSize = 1
        queueCapacity = 10
        setThreadNamePrefix("meta-ai-objects-import-")
        setWaitForTasksToCompleteOnShutdown(true)
        setAwaitTerminationSeconds(60)
    }

    @Async("metaAiObjectsImportExecutor")
    fun process(source: Path, extension: String, fileId: Long, userId: Long) {
        try {
            val parsed = reader.read(source, extension)
            val errors = parsed.errors.toMutableList()

            val keys = parsed.rows.map { it.crossgoal }.toSet()
            val mappings = if (keys.isEmpty()) {
                emptyList()
            } else {
                jiraIssueRepository.findMetaAiObjectsInitiativesByCrossgoalKeys(keys)
            }

            val initiativesByCrossgoal = mappings
                .groupBy { it.jiraKey }
                .mapValues { (_, values) -> values.map { it.agentId }.toSet() }

            val rowsByInitiative = linkedMapOf<Long, MutableList<MetaAiObjectsExcelRow>>()

            parsed.rows.forEach { row ->
                val initiativeIds = initiativesByCrossgoal[row.crossgoal].orEmpty()

                when {
                    initiativeIds.isEmpty() ->
                        errors += "Строка ${row.rowNumber}: инициатива CROSSGOAL=${row.crossgoal} не найдена"

                    initiativeIds.size != 1 ->
                        errors += "Строка ${row.rowNumber}: CROSSGOAL=${row.crossgoal} соответствует нескольким инициативам"

                    else ->
                        rowsByInitiative.getOrPut(initiativeIds.single()) { mutableListOf() }.add(row)
                }
            }

            var changedTotal = 0

            for ((initiativeId, initiativeRows) in rowsByInitiative) {
                try {
                    changedTotal += applyService.apply(initiativeId, initiativeRows, fileId, userId)
                } catch (e: Exception) {
                    log.error(
                        "Meta AI objects import failed for initiativeId={}, fileId={}",
                        initiativeId,
                        fileId,
                        e
                    )
                    errors += "Инициатива $initiativeId: ${e.message ?: "ошибка сохранения"}"
                }
            }

            if (errors.isEmpty()) {
                fileUploadStatusService.updateStatus(fileId, "success", null, userId)
                log.info("Meta AI objects import completed: fileId={}, changedObjects={}", fileId, changedTotal)
            } else {
                val description = "Ошибок: ${errors.size}. " + errors.take(20).joinToString("; ")
                fileUploadStatusService.updateStatus(fileId, "error", description, userId)
                log.warn(
                    "Meta AI objects import finished with errors: fileId={}, changedObjects={}, errors={}",
                    fileId,
                    changedTotal,
                    errors.size
                )
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

import org.apache.poi.poifs.filesystem.FileMagic
import org.slf4j.LoggerFactory
import org.springframework.core.task.TaskRejectedException
import org.springframework.http.HttpStatus
import org.springframework.stereotype.Service
import org.springframework.web.multipart.MultipartFile
import java.nio.file.Files
import java.nio.file.Path
import java.text.MessageFormat
import java.util.Locale

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

    fun upload(file: MultipartFile): Long {
        val fileName = file.originalFilename.orEmpty()
            .replace('\\', '/')
            .substringAfterLast('/')

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
                importWorker.process(source, extension, fileId, userId)
                submitted = true
                return fileId
            } catch (rejected: TaskRejectedException) {
                runCatching {
                    fileUploadStatusService.updateStatus(fileId, "error", "Очередь загрузок переполнена", userId)
                }.onFailure {
                    log.error("Could not mark rejected upload as ERROR: fileId={}", fileId, it)
                }

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
}

```
