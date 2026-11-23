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

            val crossgoalKeys = parsed.rows.map { it.crossgoal }.toSet()
            val initiativeMappings = if (crossgoalKeys.isEmpty()) {
                emptyList()
            } else {
                // Повторно используем существующий запрос старой загрузки КЭ.
                // Он возвращает ВСЕ активные инициативы, связанные с CROSSGOAL.
                jiraIssueRepository.findAgentIdsByCrossgoalKeys(crossgoalKeys)
            }

            val initiativeIdsByCrossgoal = initiativeMappings
                .groupBy { it.jiraKey }
                .mapValues { (_, mappings) -> mappings.map { it.agentId }.distinct() }

            val rowsByInitiative = linkedMapOf<Long, MutableList<MetaAiObjectsExcelRow>>()

            parsed.rows.forEach { row ->
                val initiativeIds = initiativeIdsByCrossgoal[row.crossgoal].orEmpty()

                if (initiativeIds.isEmpty()) {
                    errors += "Строка ${row.rowNumber}: инициатива CROSSGOAL=${row.crossgoal} не найдена"
                } else {
                    // В отличие от прежней проверки size == 1, передаём строку
                    // каждой найденной инициативе. Порядок строк внутри каждой
                    // инициативы сохраняется (последняя строка с КЭ побеждает).
                    initiativeIds.forEach { initiativeId ->
                        rowsByInitiative.getOrPut(initiativeId) { mutableListOf() }.add(row)
                    }
                }
            }

            var changedTotal = 0

            for ((initiativeId, initiativeRows) in rowsByInitiative) {
                try {
                    changedTotal += applyService.apply(initiativeId, initiativeRows, fileId, userId)
                } catch (exception: Exception) {
                    log.error(
                        "Meta AI objects import failed for initiativeId={}, fileId={}",
                        initiativeId,
                        fileId,
                        exception
                    )
                    errors += "Инициатива $initiativeId: ${exception.message ?: "ошибка сохранения"}"
                }
            }

            if (errors.isEmpty()) {
                fileUploadStatusService.updateStatus(fileId, "success", null, userId)
                log.info(
                    "Meta AI objects import completed: fileId={}, changedObjects={}",
                    fileId,
                    changedTotal
                )
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
        } catch (exception: Exception) {
            log.error("Meta AI objects import failed: fileId={}", fileId, exception)
            try {
                fileUploadStatusService.updateStatus(fileId, "error", exception.message, userId)
            } catch (statusException: Exception) {
                log.error("Could not update file_upload status: fileId={}", fileId, statusException)
            }
        } finally {
            try {
                Files.deleteIfExists(source)
            } catch (exception: Exception) {
                log.warn("Could not delete temporary Excel: {}", source, exception)
            }
        }
    }
}

```
