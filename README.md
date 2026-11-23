```java


/**
 * Фоновая обработка загруженного файла связанных AI-объектов.
 *
 * Разбирает строки, сопоставляет CROSSGOAL со всеми найденными инициативами,
 * применяет изменения по каждой инициативе и фиксирует статус загрузки.
 */
@Service
class MetaAiObjectsImportWorker(
    private val reader: MetaAiObjectsExcelReader,
    private val jiraIssueRepository: JiraIssueRepository,
    private val applyService: MetaAiObjectsImportApplyService,
    private val fileUploadStatusService: FileUploadStatusService
) {

    private val log = LoggerFactory.getLogger(javaClass)

    /**
     * Импорты выполняются последовательно.
     * В очереди может находиться до 10 файлов.
     */
    @Bean("metaAiObjectsImportExecutor")
    fun metaAiObjectsImportExecutor() = ThreadPoolTaskExecutor().apply {
        corePoolSize = 1
        maxPoolSize = 1
        queueCapacity = 10
        setThreadNamePrefix("meta-ai-objects-import-")
        setWaitForTasksToCompleteOnShutdown(true)
        setAwaitTerminationSeconds(60)
    }

    /**
     * Импортирует связанные AI-объекты и обновляет статус файловой загрузки.
     *
     * Один CROSSGOAL может соответствовать нескольким инициативам.
     * Все найденные инициативы обрабатываются независимо.
     *
     * Ошибки строк и инициатив накапливаются в file_upload.error_description.
     * Временный файл удаляется после завершения обработки.
     *
     * @param source временный файл загрузки
     * @param extension расширение файла (xls или xlsx)
     * @param fileId идентификатор загрузки
     * @param userId идентификатор пользователя
     */
    @Async("metaAiObjectsImportExecutor")
    fun process(source: Path, extension: String, fileId: Long, userId: Long) {
        try {
            val parsed = reader.read(source, extension)
            val errors = parsed.errors.toMutableList()

            val crossgoalKeys = parsed.rows.map { it.crossgoal }.toSet()

            val mappings = if (crossgoalKeys.isEmpty()) {
                emptyList()
            } else {
                jiraIssueRepository.findAgentIdsByCrossgoalKeys(crossgoalKeys)
            }

            val initiativeIdsByCrossgoal = mappings
                .groupBy { it.jiraKey }
                .mapValues { (_, values) -> values.map { it.agentId }.toSet() }

            val rowsByInitiative = linkedMapOf<Long, MutableList<MetaAiObjectsExcelRow>>()

            parsed.rows.forEach { row ->
                val initiativeIds = initiativeIdsByCrossgoal[row.crossgoal].orEmpty()

                if (initiativeIds.isEmpty()) {
                    errors += "Строка ${row.rowNumber}: инициатива CROSSGOAL=${row.crossgoal} не найдена"
                } else {
                    initiativeIds.forEach { initiativeId ->
                        rowsByInitiative.getOrPut(initiativeId) { mutableListOf() }.add(row)
                    }
                }
            }

            var changedTotal = 0

            for ((initiativeId, initiativeRows) in rowsByInitiative) {
                try {
                    changedTotal += applyService.apply(
                        initiativeId,
                        initiativeRows,
                        fileId,
                        userId
                    )
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
        } catch (e: Exception) {
            log.error("Meta AI objects import failed: fileId={}", fileId, e)

            try {
                fileUploadStatusService.updateStatus(fileId, "error", e.message, userId)
            } catch (statusError: Exception) {
                log.error(
                    "Could not update file_upload status: fileId={}",
                    fileId,
                    statusError
                )
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
```
