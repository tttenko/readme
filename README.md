```java


/**
     * Пакетно загружает существующие объекты для предварительного сравнения
     * с данными Excel.
     *
     * Включает неактивные записи, поскольку они могут требовать восстановления.
     */
    @Query("""
        SELECT obj
        FROM InitiativeMetaAiObjectEntity obj
        WHERE obj.initiative.id IN :initiativeIds
          AND obj.ciSolution IN :ciSolutions
    """)
    fun findExistingForImport(
        @Param("initiativeIds") initiativeIds: Collection<Long>,
        @Param("ciSolutions") ciSolutions: Collection<String>
    ): List<InitiativeMetaAiObjectEntity>

    import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Propagation
import org.springframework.transaction.annotation.Transactional

/**
 * Применение строк Excel к связанным AI-объектам инициатив.
 */
@Service
class MetaAiObjectsImportApplyService(
    private val aiAgentRepository: AIAgentRepository,
    private val metaAiObjectRepository: InitiativeMetaAiObjectRepository,
    private val historyRepository: InitiativeAiObjectHistoryRepository
) {

    /**
     * Предварительно определяет инициативы, для которых требуется изменение данных.
     *
     * Существующие объекты загружаются пакетами, после чего сравниваются
     * с конечными значениями строк Excel.
     *
     * Для инициатив без изменений последующий apply не вызывается.
     *
     * @return идентификаторы инициатив, требующих создания или обновления объектов
     */
    @Transactional(readOnly = true)
    fun findInitiativeIdsWithChanges(
        rowsByInitiative: Map<Long, List<MetaAiObjectsExcelRow>>
    ): Set<Long> {
        if (rowsByInitiative.isEmpty()) return emptySet()

        val ciSolutions = rowsByInitiative.values
            .flatten()
            .map { it.ciSolution }
            .toSet()

        val storedByKey = rowsByInitiative.keys
            .chunked(250)
            .flatMap { initiativeIds ->
                metaAiObjectRepository.findExistingForImport(initiativeIds, ciSolutions)
            }
            .associateBy { it.initiative?.id to it.ciSolution }

        return rowsByInitiative.mapNotNull { (initiativeId, rows) ->
            val finalRows = getFinalRows(rows)

            val hasChanges = finalRows.any { row ->
                val stored = storedByKey[initiativeId to row.ciSolution]
                hasChanges(stored, row)
            }

            initiativeId.takeIf { hasChanges }
        }.toSet()
    }

    /**
     * Создаёт или обновляет связанные объекты одной инициативы.
     *
     * Изменения объектов и запись истории выполняются в одной транзакции.
     * Ошибка обработки одной инициативы не откатывает остальные.
     *
     * @return количество уникальных созданных или изменённых объектов
     */
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    fun apply(
        initiativeId: Long,
        rows: List<MetaAiObjectsExcelRow>,
        fileId: Long,
        userId: Long
    ): Int {
        val initiative = aiAgentRepository.findByIdForUpdate(initiativeId)
            ?: throw IllegalArgumentException("Инициатива $initiativeId больше не существует")

        val finalRows = getFinalRows(rows)

        val storedByCi = metaAiObjectRepository.findAllWithCiSolutionByInitiativeId(initiativeId)
            .associateBy { it.ciSolution }

        var changedCount = 0

        for (row in finalRows) {
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
            } else if (hasChanges(stored, row)) {
                stored.crossgoal = row.crossgoal
                stored.name = row.name
                stored.isAgent = row.isAgent
                stored.gigausage = row.gigausage
                stored.metaLink = row.metaLink
                stored.isActive = true
                stored.updatedBy = userId.toString()

                // Сущность управляется Hibernate.
                // Dirty checking выполнит UPDATE при завершении транзакции.
                changedCount++
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

    /**
     * Возвращает конечные значения объектов.
     *
     * Если КЭ встречается несколько раз, используется последняя строка файла.
     */
    private fun getFinalRows(rows: List<MetaAiObjectsExcelRow>): Collection<MetaAiObjectsExcelRow> {
        val finalRows = linkedMapOf<String, MetaAiObjectsExcelRow>()

        rows.forEach { row ->
            finalRows[row.ciSolution] = row
        }

        return finalRows.values
    }

    /**
     * Проверяет, требуется ли создание или обновление объекта.
     *
     * Пустые дополнительные поля также участвуют в сравнении:
     * значения, отсутствующие в файле, перезаписываются на null.
     */
    private fun hasChanges(
        stored: InitiativeMetaAiObjectEntity?,
        row: MetaAiObjectsExcelRow
    ): Boolean {
        if (stored == null) return true

        return stored.crossgoal != row.crossgoal ||
                stored.name != row.name ||
                stored.isAgent != row.isAgent ||
                stored.gigausage != row.gigausage ||
                stored.metaLink != row.metaLink ||
                !stored.isActive
    }
}

import org.springframework.context.annotation.Bean
import org.springframework.scheduling.annotation.Async
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor
import org.springframework.stereotype.Service
import java.nio.file.Files
import java.nio.file.Path

/**
 * Фоновая обработка загруженного файла связанных AI-объектов.
 *
 * Разбирает строки, сопоставляет CROSSGOAL со всеми найденными инициативами,
 * определяет необходимые изменения и фиксирует статус загрузки.
 */
@Service
class MetaAiObjectsImportWorker(
    private val reader: MetaAiObjectsExcelReader,
    private val jiraIssueRepository: JiraIssueRepository,
    private val applyService: MetaAiObjectsImportApplyService,
    private val fileUploadStatusService: FileUploadStatusService
) {

    private val log by logger()

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
     * Импортирует связанные AI-объекты и обновляет статус загрузки.
     *
     * Один CROSSGOAL может соответствовать нескольким инициативам.
     * Для инициатив без изменений повторное сохранение не выполняется.
     *
     * Ошибки накапливаются в file_upload.error_description.
     * Временный файл удаляется после завершения обработки.
     */
    @Async("metaAiObjectsImportExecutor")
    fun process(source: Path, extension: String, fileId: Long, userId: Long) {
        val startedAt = System.nanoTime()

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

            // Пакетная проверка изменений до открытия транзакций сохранения.
            val checkStartedAt = System.nanoTime()

            val initiativeIdsWithChanges = applyService.findInitiativeIdsWithChanges(rowsByInitiative)

            log.info(
                "Meta AI objects precheck completed: fileId={}, totalInitiatives={}, changedInitiatives={}, elapsedMs={}",
                fileId,
                rowsByInitiative.size,
                initiativeIdsWithChanges.size,
                (System.nanoTime() - checkStartedAt) / 1_000_000
            )

            var changedTotal = 0

            for ((initiativeId, initiativeRows) in rowsByInitiative) {
                if (initiativeId !in initiativeIdsWithChanges) continue

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
                    "Meta AI objects import completed: fileId={}, totalInitiatives={}, changedInitiatives={}, changedObjects={}, elapsedMs={}",
                    fileId,
                    rowsByInitiative.size,
                    initiativeIdsWithChanges.size,
                    changedTotal,
                    (System.nanoTime() - startedAt) / 1_000_000
                )
            } else {
                val description = "Ошибок: ${errors.size}. " + errors.take(20).joinToString("; ")

                fileUploadStatusService.updateStatus(fileId, "error", description, userId)

                log.warn(
                    "Meta AI objects import finished with errors: fileId={}, changedObjects={}, errors={}, elapsedMs={}",
                    fileId,
                    changedTotal,
                    errors.size,
                    (System.nanoTime() - startedAt) / 1_000_000
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
