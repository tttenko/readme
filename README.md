```java
interface AppliedMetricRepository: JpaRepository<AppliedMetricsEntity, Long> {

    @Modifying
    @Query("DELETE FROM AppliedMetricsEntity a WHERE a.aiAgent.id = :aiAgentId")
    fun deleteAllByAiAgentId(@Param("aiAgentId") aiAgentId: Long)

    /**
     * Метрики загружаются пачкой, а импорт меняет только изменённые строки.
     * Неизменённые записи сохраняют id, created и updated.
     */
    fun findAllByAiAgentId(aiAgentId: Long): List<AppliedMetricsEntity>

    @EntityGraph(attributePaths = ["metric"])
    fun findAllByAiAgentIdIn(aiAgentIds: Collection<Long>): List<AppliedMetricsEntity>

}

interface AgentContactRepository : JpaRepository<AgentContactEntity, Long> {

    /** Оставлено для сторонних текущих потребителей; импорт больше не вызывает. */
    @Transactional
    @Modifying
    @Query(
        value = """
            delete from agent_contact ac
            using ai_agent a
            where ac.agent_id = a.id
              and a.agent_id not like '%PULT%'
              and a.import_status is distinct from 'blocked'
        """,
        nativeQuery = true
    )
    fun deleteAllForImportedNonBlockedAgents(): Int

    /** Удаляются только связи агентов, которых импорт НЕ обработал. */
    @Modifying
    @Query(value = "DELETE FROM agent_contact WHERE agent_id IN (:ids)", nativeQuery = true)
    fun deleteForAgentIds(@Param("ids") agentIds: Collection<Long>): Int
}

@Service
class DictAndProcessService(
    private val blockRepository: BlockRepository,
    private val platformRepository: PlatformRepository,
    private val programRepository: ProgramRepository,
    private val processRepository: ProcessRepository,
    private val messageProvider: MessageProvider
) {
    private val log by logger()

    /** Применение только во внешней бизнес-транзакции. */
    @Transactional(propagation = Propagation.MANDATORY)
    fun applyPreparedDictionaries(
        workbook: PreparedExcelWorkbook,
        fileUploadId: Long,
        userId: Long,
        terBanksRefs: MutableList<TerBank>?
    ): DictAndProcessContext = doUpdateDictionaries(workbook, fileUploadId, userId, terBanksRefs)

    private fun doUpdateDictionaries(
        workbook: PreparedExcelWorkbook,
        fileUploadId: Long,
        userId: Long,
        terBanksRefs: MutableList<TerBank>?
    ): DictAndProcessContext {
        val operationDetails = "Request 'process DICT and PROCESSES pages' with user=$userId "
        log.logBefore(operationDetails)

        val availablePlatforms = platformRepository.findAll().toMutableList()
        val availablePrograms = programRepository.findAll().toMutableList()
        val availableProcesses = processRepository.findAll().toMutableList()

        val readDictData = readDictData(workbook)
        val divisionsDict = readDictData.divisions
        val platformsDict = readDictData.platforms
        val blocksDict = readDictData.blocks
        val programsDict = readDictData.programms
        val processData = readProcessData(
            workbook = workbook,
            blockCodes = blocksDict.map { it.code }.toList(),
            programCodes = programsDict.map { it.code }.toList()
        )

        updatePlatformDict(platformsDict, availablePlatforms)
        updatePrograms(programsDict, availablePrograms)
        val actualBlocks = blockRepository.findAll().toMutableList()
        updateProcesses(
            processDict = processData.first,
            availableProcesses = availableProcesses,
            availablePrograms = availablePrograms,
            availableBlocks = actualBlocks
        )

        return DictAndProcessContext(
            divisionsDict = divisionsDict,
            blocksDict = blocksDict,
            fileUploadId = fileUploadId, // Используем переданный fileUploadId
            processData = processData,
            terBanksRefs = terBanksRefs,
        ).also { log.logSuccess(operationDetails) }
    }

    private fun readDictData(workbook: PreparedExcelWorkbook): DictData {
        val divisions = mutableListOf<DivisionsDto>()
        val platforms = mutableListOf<PlatformsDto>()
        val blocks = mutableListOf<BlocksDto>()
        val programms = mutableListOf<ProgramDto>()

        val sheet = workbook.getSheet(DICT)
        return sheet.readRows { iterator ->
            val columnHeaders = mutableMapOf<Int, String?>()
            var currentRowIndex = 0

            // Собираем заголовки
            while (iterator.hasNext()) {
                val row = iterator.next()
                if (currentRowIndex == DICT_HEADER_ROW_INDEX) {
                    for (cell in row) {
                        columnHeaders[cell.columnIndex] = cell.stringCellValue?.trim()
                    }
                    break
                }
                currentRowIndex++
            }

            // Ищем колонки в пределах заданных секций
            val tribeLabelCol = findColumnInRange(columnHeaders, DIVISIONS_SECTION_START..DIVISIONS_SECTION_END, DICT_LABEL_SUBHEADER)
            val tribeShortNameCol = findColumnInRange(columnHeaders, DIVISIONS_SECTION_START..DIVISIONS_SECTION_END, DICT_SHORT_NAME_SUBHEADER)
            val tribeCodeCol = findColumnInRange(columnHeaders, DIVISIONS_SECTION_START..DIVISIONS_SECTION_END, DICT_CODE_SUBHEADER)
            val tribeBlockCol = findColumnInRange(columnHeaders, DIVISIONS_SECTION_START..DIVISIONS_SECTION_END, DICT_BLOCK_SUBHEADER)
            val tribeNameCol = findColumnInRange(columnHeaders, DIVISIONS_SECTION_START..DIVISIONS_SECTION_END, DICT_NAME_SUBHEADER)

            val surfaceNameCol = findColumnInRange(columnHeaders, PLATFORMS_SECTION_START..PLATFORMS_SECTION_END, DICT_NAME_SUBHEADER)
            val surfaceCodeCol = findColumnInRange(columnHeaders, PLATFORMS_SECTION_START..PLATFORMS_SECTION_END, DICT_CODE_SUBHEADER)

            val blockCodeCol = findColumnInRange(columnHeaders, BLOCKS_SECTION_START..BLOCKS_SECTION_END, DICT_CODE_SUBHEADER)
            val blockNameCol = findColumnInRange(columnHeaders, BLOCKS_SECTION_START..BLOCKS_SECTION_END, DICT_NAME_SUBHEADER)
            val blockCuratorNameCol = findColumnInRange(columnHeaders, BLOCKS_SECTION_START..BLOCKS_SECTION_END, DICT_CURATOR_NAME_SUBHEADER)
            val blockShortNameCol = findColumnInRange(columnHeaders, BLOCKS_SECTION_START..BLOCKS_SECTION_END, DICT_SHORTNAME_BLOCKS_SUBHEADER)

            val programCodeCol = findColumnInRange(columnHeaders, PROGRAMS_SECTION_START..PROGRAMS_SECTION_END, DICT_CODE_SUBHEADER)
            val programFileNameCol = findColumnInRange(columnHeaders, PROGRAMS_SECTION_START..PROGRAMS_SECTION_END, DICT_FILENAME_PROGRAM_SUBHEADER)
            val programNameCol = findColumnInRange(columnHeaders, PROGRAMS_SECTION_START..PROGRAMS_SECTION_END, DICT_NAME_SUBHEADER)

            // Проверяем найденные колонки
            val missingColumns = checkRequiredColumns(
                tribeLabelCol, tribeShortNameCol, tribeCodeCol, tribeBlockCol, tribeNameCol,
                surfaceNameCol, surfaceCodeCol,
                blockShortNameCol, blockNameCol, blockCodeCol, blockCuratorNameCol,
                programCodeCol, programFileNameCol, programNameCol
            )

            if (missingColumns.isNotEmpty()) {
                throw AiResponseException(
                    status = HttpStatus.BAD_REQUEST,
                    errorCode = AI_UPLOAD_EMPTY_DICT_FILE,
                    message = "Missing columns in DICT file: ${missingColumns.joinToString()}"
                )
            }

            // Обрабатываем данные
            while (iterator.hasNext()) {
                val row = iterator.next()
                currentRowIndex++

                if (currentRowIndex >= DICT_DATA_START_ROW_INDEX) {
                    processDictRow(
                        row = row,
                        tribeLabelCol = tribeLabelCol,
                        tribeShortNameCol = tribeShortNameCol,
                        tribeCodeCol = tribeCodeCol,
                        tribeBlockCol = tribeBlockCol,
                        tribeNameCol = tribeNameCol,
                        surfaceNameCol = surfaceNameCol,
                        surfaceCodeCol = surfaceCodeCol,
                        blockCodeCol = blockCodeCol,
                        blockNameCol = blockNameCol,
                        blockCuratorNameCol = blockCuratorNameCol,
                        blockShortNameCol = blockShortNameCol,
                        programCodeCol = programCodeCol,
                        programFileNameCol = programFileNameCol,
                        programNameCol = programNameCol,
                        divisions = divisions,
                        platforms = platforms,
                        blocks = blocks,
                        programms = programms
                    )
                }
            }

            DictData(divisions = divisions, platforms = platforms, blocks = blocks, programms = programms)
        }
    }

    private fun findColumnInRange(columnHeaders: Map<Int, String?>, range: IntRange, headerName: String): Int {
        return range.firstOrNull { index ->
            columnHeaders[index]?.equals(headerName, ignoreCase = true) == true
        } ?: -1
    }

    private fun checkRequiredColumns(vararg columns: Int): List<String> {
        val missing = mutableListOf<String>()
        val columnNames = arrayOf(
            "tribeLabel", "tribeShortName", "tribeCode", "tribeBlock", "tribeName",
            "surfaceName", "surfaceCode",
            "blockShortName", "blockName", "blockCode", "blockCuratorName",
            "programCode", "programFileName", "programName"
        )

        columns.forEachIndexed { index, col ->
            if (col == -1) {
                missing.add(columnNames[index])
            }
        }
        return missing
    }

    private fun processDictRow(
        row: PreparedRow,
        tribeLabelCol: Int,
        tribeShortNameCol: Int,
        tribeCodeCol: Int,
        tribeBlockCol: Int,
        tribeNameCol: Int,
        surfaceNameCol: Int,
        surfaceCodeCol: Int,
        blockCodeCol: Int,
        blockNameCol: Int,
        blockCuratorNameCol: Int,
        blockShortNameCol: Int,
        programCodeCol: Int,
        programFileNameCol: Int,
        programNameCol: Int,
        divisions: MutableList<DivisionsDto>,
        platforms: MutableList<PlatformsDto>,
        blocks: MutableList<BlocksDto>,
        programms: MutableList<ProgramDto>
    ) {
        val tribeLabel = row.getCell(tribeLabelCol)?.stringCellValue ?: ""
        val shortName = row.getCell(tribeShortNameCol)?.stringCellValue ?: ""
        val tribeCode = row.getCell(tribeCodeCol)?.stringCellValue ?: ""
        val tribeBlock = row.getCell(tribeBlockCol)?.stringCellValue ?: ""
        val tribeName = row.getCell(tribeNameCol)?.stringCellValue ?: ""
        if (tribeLabel.isNotBlank() || shortName.isNotBlank() || tribeCode.isNotBlank() || tribeBlock.isNotBlank() || tribeName.isNotBlank()) {
            divisions.add(DivisionsDto(tribeLabel, shortName, tribeCode, tribeBlock, tribeName))
        }

        val surfaceName = row.getCell(surfaceNameCol)?.stringCellValue ?: ""
        val surfaceCode = row.getCell(surfaceCodeCol)?.stringCellValue ?: ""
        if (surfaceName.isNotBlank() || surfaceCode.isNotBlank()) {
            platforms.add(PlatformsDto(surfaceName, surfaceCode))
        }

        val blockShortName = row.getCell(blockShortNameCol)?.stringCellValue ?: ""
        val blockCode = row.getCell(blockCodeCol)?.stringCellValue ?: ""
        val blockName = row.getCell(blockNameCol)?.stringCellValue ?: ""
        val blockCurator = row.getCell(blockCuratorNameCol)?.stringCellValue ?: ""
        if (blockShortName.isNotBlank() || blockCode.isNotBlank() || blockName.isNotBlank() || blockCurator.isNotBlank()) {
            blocks.add(BlocksDto(blockCode, blockShortName, blockName, blockCurator))
        }

        val programFileName = row.getCell(programFileNameCol)?.stringCellValue ?: ""
        val programCode = row.getCell(programCodeCol)?.stringCellValue ?: ""
        val programName = row.getCell(programNameCol)?.stringCellValue ?: ""
        if (programFileName.isNotBlank() || programCode.isNotBlank() || programName.isNotBlank()) {
            programms.add(
                ProgramDto(
                    fileName = programFileName,
                    code = programCode,
                    name = programName
                )
            )
        }
    }

    private fun readProcessData(
        workbook: PreparedExcelWorkbook,
        blockCodes: List<String>,
        programCodes: List<String>
    ): Pair<List<ProcessDto>, List<ProcessesMappingDto>> {
        val processes = mutableListOf<ProcessDto>()
        val processesMapping = mutableListOf<ProcessesMappingDto>()

        val sheet = workbook.getSheet(PROCESSES)
        return sheet.readRows { iterator ->
            val columnHeaders = mutableMapOf<Int, String?>()
            var currentRowIndex = 0

            // Собираем заголовки
            while (iterator.hasNext()) {
                val row = iterator.next()
                if (currentRowIndex == DICT_HEADER_ROW_INDEX) {
                    for (cell in row) {
                        columnHeaders[cell.columnIndex] = cell.stringCellValue?.trim()
                    }
                    break
                }
                currentRowIndex++
            }

            // Ищем колонки в пределах секций процессов
            val processShortNameCol = findColumnInRange(columnHeaders, PROCESSES_SECTION_START..PROCESSES_SECTION_END, DICT_SHORTNAME_BLOCKS_SUBHEADER)
            val processNameCol = findColumnInRange(columnHeaders, PROCESSES_SECTION_START..PROCESSES_SECTION_END, DICT_NAME_SUBHEADER)
            val processBlockCol = findColumnInRange(columnHeaders, PROCESSES_SECTION_START..PROCESSES_SECTION_END, DICT_BLOCK_SUBHEADER)
            val processProgramCol = findColumnInRange(columnHeaders, PROCESSES_SECTION_START..PROCESSES_SECTION_END, PROCESS_PROGRAM_SUBHEADER)

            // Ищем колонки в пределах секций маппинга процессов
            val processMappingDpssCol = findColumnInRange(columnHeaders, PROCESSES_MAPPINGS_SECTION_START..PROCESSES_MAPPINGS_SECTION_END, PROCESS_MAPPINGS_CODE_DPSS_SUBHEADER)
            val processMappingArisCol = findColumnInRange(columnHeaders, PROCESSES_MAPPINGS_SECTION_START..PROCESSES_MAPPINGS_SECTION_END, PROCESS_MAPPINGS_CODE_ARIS_SUBHEADER)

            // Проверяем найденные колонки
            val missingColumns = mutableListOf<String>()
            if (processShortNameCol == -1) missingColumns.add("processShortName")
            if (processNameCol == -1) missingColumns.add("processName")
            if (processBlockCol == -1) missingColumns.add("processBlock")
            if (processProgramCol == -1) missingColumns.add("processProgram")
            if (processMappingDpssCol == -1) missingColumns.add("processMappingDpss")
            if (processMappingArisCol == -1) missingColumns.add("processMappingAris")

            if (missingColumns.isNotEmpty()) {
                throw AiResponseException(
                    status = HttpStatus.BAD_REQUEST,
                    errorCode = AI_UPLOAD_EMPTY_DICT_FILE,
                    message = "Missing columns in PROCESSES file: ${missingColumns.joinToString()}"
                )
            }

            // Обрабатываем данные
            while (iterator.hasNext()) {
                val row = iterator.next()
                currentRowIndex++

                if (currentRowIndex >= DICT_DATA_START_ROW_INDEX) {
                    val processShortName = row.getCell(processShortNameCol)?.stringCellValue ?: ""
                    val processProgram = row.getCell(processProgramCol)?.stringCellValue ?: ""
                    val processBlock = row.getCell(processBlockCol)?.stringCellValue ?: ""
                    val processName = row.getCell(processNameCol)?.stringCellValue ?: ""

                    if (processShortName.isNotBlank() || processProgram.isNotBlank() || processBlock.isNotBlank() || processName.isNotBlank()) {
                        if (!blockCodes.contains(processBlock)) {
                            throw AiResponseException(
                                status = HttpStatus.BAD_REQUEST,
                                errorCode = AI_PROCESS_NOT_FOUND_BLOCK,
                                message = MessageFormat.format(messageProvider[AI_PROCESS_NOT_FOUND_BLOCK], processShortName)
                            )
                        }
                        if (processProgram.isNotBlank() && !programCodes.contains(processProgram)) {
                            throw AiResponseException(
                                status = HttpStatus.BAD_REQUEST,
                                errorCode = AI_PROCESS_NOT_FOUND_PROGRAM,
                                message = MessageFormat.format(messageProvider[AI_PROCESS_NOT_FOUND_PROGRAM], processShortName)
                            )
                        }

                        processes.add(
                            ProcessDto(
                                shortName = processShortName,
                                program = processProgram,
                                block = processBlock,
                                name = processName,
                                rowNum = row.rowNum
                            )
                        )
                    }

                    val processMappingDpss = row.getCell(processMappingDpssCol)?.stringCellValue ?: ""
                    val processMappingAris = row.getCell(processMappingArisCol)?.stringCellValue ?: ""
                    if (processMappingDpss.isNotBlank() || processMappingAris.isNotBlank()) {
                        processesMapping.add(
                            ProcessesMappingDto(
                                codeDpss = processMappingDpss,
                                codeAris = processMappingAris,
                                rowNum = row.rowNum
                            )
                        )
                    }
                }
            }

            Pair(processes, processesMapping)
        }
    }

    private fun updatePrograms(programsDict: List<ProgramDto>, availablePrograms: MutableList<ProgramEntity>) {
        val platformUpdateDuration = measureTimeMillis {
            // Старый поиск firstOrNull выбирал ПЕРВОЕ совпадение.
            // Существующий код не trim-ится: это важно для точной совместимости.
            val programsByCode = linkedMapOf<String?, ProgramEntity>()
            availablePrograms.forEach { program ->
                programsByCode.putIfAbsent(program.code?.lowercase(), program)
            }
            programsDict.forEach { programFromDict ->
                programsByCode[programFromDict.code.trim().lowercase()]
                    ?.let { existingEntity ->
                        existingEntity.name = programFromDict.name.trim()
                        existingEntity.fileName = programFromDict.fileName.trim()
                        existingEntity.disabled = false
                        // Managed-сущность сохранится через dirty checking при flush/commit.
                    } ?: programRepository.save(
                    ProgramEntity().apply {
                        fileName = programFromDict.fileName.trim()
                        name = programFromDict.name.trim()
                        code = programFromDict.code.trim()
                    }).also { availablePrograms.add(it); programsByCode[programFromDict.code.trim().lowercase()] = it }
            }
        }
        disableEntitiesByCode(codesInFile = programsDict.map { it.code }.toSet(),
            allEntities = availablePrograms,
            codeExtractor = { it.code },
            isDisabled = { it.disabled },
            setDisabled = { entity, disabled -> entity.disabled = disabled })
        log.debug("Operation update programs from dict took $platformUpdateDuration ms")
    }

    private fun updateProcesses(
        processDict: List<ProcessDto>,
        availableProcesses: MutableList<ProcessEntity>,
        availablePrograms: MutableList<ProgramEntity>,
        availableBlocks: MutableList<BlockEntity>
    ) {
        val platformUpdateDuration = measureTimeMillis {
            // firstOrNull по исходному списку -> первая запись выигрывает.
            val processesByName = linkedMapOf<String?, ProcessEntity>()
            availableProcesses.forEach { process ->
                processesByName.putIfAbsent(process.shortName?.lowercase(), process)
            }
            val blocksByCode = linkedMapOf<String?, BlockEntity>()
            availableBlocks.forEach { block -> blocksByCode.putIfAbsent(block.code, block) }
            val programsByCode = linkedMapOf<String?, ProgramEntity>()
            availablePrograms.forEach { program -> programsByCode.putIfAbsent(program.code, program) }
            processDict.forEach { processFromDict ->
                if (!blocksByCode.containsKey(processFromDict.block)) return@forEach
                processesByName[processFromDict.shortName.trim().lowercase()]
                    ?.let { existingEntity ->
                        existingEntity.name = processFromDict.name.trim()
                        existingEntity.program = programsByCode[processFromDict.program]
                        existingEntity.block = blocksByCode[processFromDict.block]
                        existingEntity.disabled = false
                        // Managed-сущность сохранится через dirty checking при flush/commit.
                    } ?: processRepository.save(
                    ProcessEntity().apply {
                        shortName = processFromDict.shortName
                        name = processFromDict.name.trim()
                        program = programsByCode[processFromDict.program]
                        block = blocksByCode[processFromDict.block]
                    }).also {
                        availableProcesses.add(it)
                        processesByName.putIfAbsent(it.shortName?.lowercase(), it)
                    }
            }
        }
        disableEntitiesByCode(codesInFile = processDict.map { it.shortName }.toSet(),
            allEntities = availableProcesses,
            codeExtractor = { it.shortName },
            isDisabled = { it.disabled },
            setDisabled = { entity, disabled -> entity.disabled = disabled })
        log.debug("Operation update processes from dict took $platformUpdateDuration ms")
    }

    private fun <ENTITY : Any> disableEntitiesByCode(
        codesInFile: Set<String?>,
        allEntities: MutableList<ENTITY>,
        codeExtractor: (ENTITY) -> String?,
        isDisabled: (ENTITY) -> Boolean?,
        setDisabled: (ENTITY, Boolean) -> Unit
    ) {
        allEntities.forEach { entity ->
            val shouldDisable = codeExtractor(entity) !in codesInFile
            if (isDisabled(entity) != shouldDisable) {
                setDisabled(entity, shouldDisable)
                // managed entity: Hibernate dirty-checking сохранит изменения при flush
            }
        }
    }

    private fun updatePlatformDict(platformsDict: List<PlatformsDto>, availablePlatforms: MutableList<PlatformEntity>) {
        val platformUpdateDuration = measureTimeMillis {
            // Не использовать associateBy(): при дублирующихся кодах он выбирает
            // последнюю запись, тогда как старый firstOrNull выбирал первую.
            val platformsByCode = linkedMapOf<String?, PlatformEntity>()
            availablePlatforms.forEach { platform ->
                platformsByCode.putIfAbsent(platform.code?.lowercase(), platform)
            }
            platformsDict.forEach { platformFromDict ->
                platformsByCode[platformFromDict.code.trim().lowercase()]
                    ?.let { existingEntity ->
                        availablePlatforms.filter { it.code != existingEntity.code }
                            .firstOrNull { it.name.equals(platformFromDict.name.trim(), ignoreCase = true) }?.let {
                                throw AiResponseException(
                                    status = HttpStatus.BAD_REQUEST,
                                    errorCode = AI_UPLOAD_PLATFORM_NAME_DUPLICATE,
                                    message = MessageFormat.format(
                                        messageProvider[AI_UPLOAD_PLATFORM_NAME_DUPLICATE],
                                        platformFromDict.name.trim(),
                                        it.code
                                    )
                                )
                            }
                        existingEntity.name = platformFromDict.name.trim()
                        existingEntity.disabled = false
                        // Managed-сущность сохранится через dirty checking при flush/commit.
                    } ?: run {
                    availablePlatforms.firstOrNull { it.name.equals(platformFromDict.name.trim(), ignoreCase = true) }
                        ?.let {
                            throw AiResponseException(
                                status = HttpStatus.BAD_REQUEST,
                                errorCode = AI_UPLOAD_PLATFORM_NAME_DUPLICATE,
                                message = MessageFormat.format(
                                    messageProvider[AI_UPLOAD_PLATFORM_NAME_DUPLICATE],
                                    platformFromDict.name.trim(),
                                    it.code
                                )
                            )
                        }
                    platformRepository.save(
                        PlatformEntity().apply {
                            name = platformFromDict.name.trim()
                            code = platformFromDict.code.trim()
                        }).also { availablePlatforms.add(it); platformsByCode[platformFromDict.code.trim().lowercase()] = it }
                }
            }
        }
        disableEntitiesByCode(codesInFile = platformsDict.map { it.code }.toSet(),
            allEntities = availablePlatforms,
            codeExtractor = { it.code },
            isDisabled = { it.disabled },
            setDisabled = { entity, disabled -> entity.disabled = disabled }
        )
        log.debug("Operation update platform from dict took $platformUpdateDuration ms")
    }


}

class PreparedExcelWorkbook(private val sheets: Map<String, PreparedSheet>, private val paths: List<Path>) : Closeable {

    fun getSheet(name: String): PreparedSheet = sheets[name]
        ?: throw IllegalArgumentException("Missing required Excel sheet: $name")

    override fun close() {
        paths.forEach { path -> runCatching { Files.deleteIfExists(path) } }
    }

    companion object {
        // Не вводим дополнительный пользовательский лимит ячеек:
        // исходный импорт ограничивал файл 20 МБ и число строк листа агентов.
        fun create(source: Path, requiredSheetNames: List<String>): PreparedExcelWorkbook {
            val sheetPaths = mutableListOf<Path>()
            val result = linkedMapOf<String, PreparedSheet>()
            try {
                Files.newInputStream(source).use { sourceStream ->
                    StreamingReader.builder().rowCacheSize(100).bufferSize(4096).open(sourceStream).use { workbook ->
                        requiredSheetNames.distinct().forEach { name ->
                            val sheet = workbook.getSheet(name)
                                ?: throw IllegalArgumentException("Missing required Excel sheet: $name")
                            val stagePath = Files.createTempFile("excel-import-rows-", ".stage")
                            sheetPaths.add(stagePath)
                            var rowCount = 0
                            ObjectOutputStream(BufferedOutputStream(Files.newOutputStream(stagePath))).use { output ->
                                sheet.forEach { row ->
                                    val stagedCells = row.map { cell ->
                                        PreparedCell.capture(cell)
                                    }
                                    output.writeObject(PreparedRow(row.rowNum, stagedCells))
                                    output.reset() // Не удерживаем ссылки на прежние строки
                                    rowCount++
                                }
                            }
                            result[name] = PreparedSheet(stagePath, rowCount)
                        }
                    }
                }
                return PreparedExcelWorkbook(result, sheetPaths)
            } catch (failure: Exception) {
                sheetPaths.forEach { runCatching { Files.deleteIfExists(it) } }
                throw failure
            }
        }
    }
}

@Service
class ExcelUploadService(
    private val localFileTypeValidator: LocalFileTypeValidator,
    private val messageProvider: MessageProvider,
    private val fileUploadRepository: FileUploadRepository,
    private val fileUploadStatusService: FileUploadStatusService,
    private val registrationService: ExcelImportRegistrationService,
    private val preparationService: ExcelImportPreparationService,
    private val applicationService: ExcelImportApplyService,
    private val uploadPermissionService: UploadPermissionService,
    private val memoryMonitor: MemoryMonitor
) {
    private val log by logger()
    private val executor = ThreadPoolExecutor(
        1, 1, 0L, TimeUnit.MILLISECONDS,
        SynchronousQueue(),
        { task -> Thread(task, "excel-import-worker").apply { isDaemon = false } },
        ThreadPoolExecutor.AbortPolicy()
    )

    fun getFileUploadInfo(id: Long): FileUploadInfoResponse = FileUploadInfoResponse(
        fileUploadRepository.findByIdOrNull(id) ?: throw AiNotFoundException(
            errorCode = FILE_UPLOAD_NOT_FOUND,
            message = messageProvider[FILE_UPLOAD_NOT_FOUND]
        )
    )

    /** Здесь НЕТ @Transactional: нельзя запускать worker до commit регистрации. */
    fun upload(file: MultipartFile, userId: Long): Long {
        // Как раньше: если слот занят, вернуть 403 ещё до проверки типа и содержимого файла.
        // Это только пред-проверка для прежнего HTTP-контракта; реальный CAS — в register().
        if (!uploadPermissionService.isUploadAllowed()) {
            throw AiResponseException(
                HttpStatus.FORBIDDEN,
                FILE_UPLOAD_NOT_AllOWED,
                message = messageProvider[FILE_UPLOAD_NOT_AllOWED]
            )
        }
        if (!localFileTypeValidator.isFileTypeValid(file)) {
            throw PrmResponseException(HttpStatus.BAD_REQUEST, FileNotValid, messageProvider[FileNotValid])
        }
        if (file.size > 20L * 1024 * 1024) {
            throw AiResponseException(
                HttpStatus.BAD_REQUEST,
                AI_UPLOAD_MAX_FILE_SIZE,
                MessageFormat.format(messageProvider[AI_UPLOAD_MAX_FILE_SIZE])
            )
        }

        val source = java.nio.file.Files.createTempFile("excel-upload-source-", ".xlsx")
        try {
            // Диск, а не byte[] размера Excel внутри длительной задачи.
            file.inputStream.use { input ->
                java.nio.file.Files.newOutputStream(source).use { output -> input.copyTo(output) }
            }
            validateAgentRowCount(source)
            val registered = registrationService.register(file.originalFilename.orEmpty(), userId)
            try {
                executor.execute { processAsync(source, registered) }
            } catch (rejected: RejectedExecutionException) {
                // Регистрация уже зафиксирована, поэтому ERROR — отдельной транзакцией.
                runCatching {
                    fileUploadStatusService.updateStatus(registered.fileUploadId, ERROR, "Import executor rejected task", registered.userId)
                }.onFailure { log.error("Could not set ERROR after rejected import", it) }
                runCatching { uploadPermissionService.release(registered.acquiredAt, registered.userId) }
                    .onFailure { log.error("Could not release rejected import lock", it) }
                throw rejected
            }
            return registered.fileUploadId
        } catch (failure: Exception) {
            // Worker уже получил путь и сам удалит его при успешном execute();
            // здесь файл удаляется лишь при синхронной ошибке регистрации/валидации/submit.
            // Выход из метода после execute() происходит раньше асинхронной обработки,
            // и в успешном случае сюда не попадаем.
            runCatching { java.nio.file.Files.deleteIfExists(source) }
            throw failure
        }
    }

    private fun validateAgentRowCount(path: Path) {
        try {
            java.nio.file.Files.newInputStream(path).use { source ->
                StreamingReader.builder().rowCacheSize(100).bufferSize(4096).open(source).use { workbook ->
                    val sheet = workbook.getSheet(AGENTS_SHEET_NAME)
                        ?: throw IllegalArgumentException("Missing agents sheet")
                    var rows = 0
                    val iterator = sheet.iterator()
                    while (iterator.hasNext()) {
                        iterator.next()
                        if (++rows > 10_000) {
                            throw AiResponseException(
                                HttpStatus.BAD_REQUEST,
                                AI_UPLOAD_MAX_ROWS_SIZE,
                                MessageFormat.format(messageProvider[AI_UPLOAD_MAX_ROWS_SIZE])
                            )
                        }
                    }
                    if (DATA_START_ROW_INDEX >= rows - 1) {
                        throw AiResponseException(
                            HttpStatus.BAD_REQUEST,
                            AI_UPLOAD_EMPTY_FILE,
                            MessageFormat.format(messageProvider[AI_UPLOAD_EMPTY_FILE])
                        )
                    }
                }
            }
        } catch (known: AiResponseException) {
            throw known
        } catch (failure: Exception) {
            throw AiResponseException(
                HttpStatus.BAD_REQUEST,
                AI_UPLOAD_CORRUPTED_FILE,
                MessageFormat.format(messageProvider[AI_UPLOAD_CORRUPTED_FILE])
            )
        }
    }

    private fun processAsync(source: Path, registration: RegisteredExcelImport) {
        try {
            preparationService.prepare(source).use { prepared ->
                // ВАЖНО: apply — отдельный Spring bean, иначе @Transactional не сработает.
                applicationService.apply(prepared, registration)
            }
            log.info("Excel import completed successfully: fileUploadId={}", registration.fileUploadId)
        } catch (failure: Exception) {
            log.error("Excel import failed: fileUploadId={}", registration.fileUploadId, failure)
            try {
                fileUploadStatusService.updateStatus(
                    registration.fileUploadId, ERROR, failure.message, registration.userId
                )
            } catch (statusFailure: Exception) {
                log.error("Unable to update failed Excel import status", statusFailure)
            }
        } finally {
            runCatching { uploadPermissionService.release(registration.acquiredAt, registration.userId) }
                .onFailure { log.error("Unable to release owned Excel import lock", it) }
            runCatching { java.nio.file.Files.deleteIfExists(source) }
                .onFailure { log.warn("Unable to delete temporary Excel file", it) }
            memoryMonitor.logMemoryUsage("- IMPORT FINISHED")
        }
    }

    /** Контракт старого контроллера сохранён; принудительный unlock инвалидирует token worker. */
    fun unlockFileUpload(userId: Long) {
        uploadPermissionService.administrativeUnlock(userId)
    }

    @PreDestroy
    fun shutdown() {
        executor.shutdown()
    }
}

@Service
class UploadPermissionService(
    private val permissionRepository: UploadPermissionRepository
) {

    /**
     * Выполнить внутри транзакции регистрации file_upload.
     *
     * @return timestamp-токen захвата либо `null`, если слот занят (403).
     */
    @Transactional(propagation = Propagation.MANDATORY)
    fun acquire(userId: Long): LocalDateTime? {
        // PostgreSQL TIMESTAMP хранит микросекунды; округляем до той же точности.
        val acquiredAt = LocalDateTime.now().truncatedTo(ChronoUnit.MICROS)
        if (!permissionRepository.existsById("GLOBAL")) {
            // Строка GLOBAL создаётся лениво, как в исходной схеме: первый импорт
            // на чистой БД инициализирует блокировку, а не падает. Без новых миграций.
            permissionRepository.saveAndFlush(
                UploadPermission(
                    id = "GLOBAL",
                    uploadAllowed = true,
                    lastUpdated = acquiredAt
                )
            )
        }
        return when (permissionRepository.tryAcquire(acquiredAt, userId)) {
            1 -> acquiredAt
            0 -> if (permissionRepository.existsById("GLOBAL")) null
            else throw IllegalStateException("More than one GLOBAL upload permission row")
            else -> throw IllegalStateException("More than one GLOBAL upload permission row")
        }
    }

    /**
     * Вызвать последним DB-действием основной apply-транзакции.
     */
    @Transactional(propagation = Propagation.MANDATORY)
    fun confirmBeforeCommit(acquiredAt: LocalDateTime) {
        if (permissionRepository.stillOwnerForUpdate(acquiredAt) != true) {
            throw IllegalStateException("Excel import lost GLOBAL ownership before commit")
        }
    }

    /**
     * Отдельная короткая транзакция после commit/rollback; чужой lock не освобождает.
     */
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    fun release(acquiredAt: LocalDateTime, userId: Long) {
        permissionRepository.releaseIfOwner(acquiredAt, userId)
    }

    /** Быстрая проверка для сохранения старого приоритета ошибки 403 над ошибкой файла.
     * Атомарное получение блокировки происходит позже внутри register().
     */
    @Transactional(readOnly = true)
    fun isUploadAllowed(): Boolean = permissionRepository.readPermissionFlag() ?: true

    /** Прежний контракт ручной разблокировки без новых recovery-шедулеров. */
    @Transactional
    fun administrativeUnlock(userId: Long) {
        if (permissionRepository.forceReleaseForAdmin(userId) != 1) {
            throw IllegalStateException("GLOBAL upload permission row is missing")
        }
    }
}

@Repository
interface UploadPermissionRepository : JpaRepository<UploadPermission, String> {

    /**
     * Атомарный CAS-захват глобального права на импорт: true→false.
     * `last_updated` — микросекундный token владения текущей загрузкой.
     */
    @Modifying
    @Query(
        value = """
            UPDATE upload_permission
            SET upload_allowed = false,
                last_updated = :acquiredAt,
                updated_by = :userId
            WHERE id = 'GLOBAL' AND upload_allowed = true
        """,
        nativeQuery = true
    )
    fun tryAcquire(@Param("acquiredAt") acquiredAt: LocalDateTime, @Param("userId") userId: Long): Int

    /**
     * Условное освобождение права только владельцем по timestamp-токену.
     */
    @Modifying
    @Query(
        value = """
            UPDATE upload_permission
            SET upload_allowed = true,
                last_updated = clock_timestamp(),
                updated_by = :userId
            WHERE id = 'GLOBAL' AND upload_allowed = false
              AND last_updated = :acquiredAt
        """,
        nativeQuery = true
    )
    fun releaseIfOwner(@Param("acquiredAt") acquiredAt: LocalDateTime, @Param("userId") userId: Long): Int

    /**
     * FOR UPDATE держит row lock до COMMIT основной транзакции.
     * Проверяется и флаг, и timestamp-токен, чтобы старый worker не закоммитил
     * после ручной разблокировки и запуска следующего worker.
     */
    @Query(
        value = """
            SELECT (upload_allowed = false AND last_updated = :acquiredAt)
            FROM upload_permission
            WHERE id = 'GLOBAL'
            FOR UPDATE
        """,
        nativeQuery = true
    )
    fun stillOwnerForUpdate(@Param("acquiredAt") acquiredAt: LocalDateTime): Boolean?

    /**
     * Публичный административный unlock работает как раньше: принудительно открывает слот.
     * Текущий worker перед COMMIT обязательно обнаружит потерю владения и откатится.
     * Вызывать вручную только после проверки, что активный импорт не выполняется.
     */
    @Modifying
    @Query(
        value = """
            UPDATE upload_permission
            SET upload_allowed = true,
                last_updated = clock_timestamp(),
                updated_by = :userId
            WHERE id = 'GLOBAL'
        """,
        nativeQuery = true
    )
    fun forceReleaseForAdmin(@Param("userId") userId: Long): Int

    @Query(value = "SELECT upload_allowed FROM upload_permission WHERE id='GLOBAL'", nativeQuery = true)
    fun readPermissionFlag(): Boolean?
}

internal class ExcelImportIntegrationTest : AbstractIntegrationTest() {

    @Autowired private lateinit var excelUploadService: ExcelUploadService
    @Autowired private lateinit var registrationService: ExcelImportRegistrationService
    @Autowired private lateinit var permissionService: UploadPermissionService
    @Autowired private lateinit var fileUploadRepository: FileUploadRepository
    @Autowired private lateinit var aiAgentRepository: AIAgentRepository
    @Autowired private lateinit var appliedMetricRepository: AppliedMetricRepository
    @Autowired private lateinit var jdbcTemplate: JdbcTemplate

    @MockkBean(relaxed = true)
    private lateinit var internalTerBanksApiClient: InternalTerBanksApiClient

    private val createdUploadIds = mutableSetOf<Long>()
    private val asyncUploadIds = mutableSetOf<Long>()

    private companion object {
        const val TEST_USER_ID = 1L
        const val TEST_BLOCK_CODE = "integration-block"
        const val TEST_BLOCK_NAME = "КИБ"
        const val FIXTURE_PATH = "src/test/resources/integration/xlsx/agents.xlsx"
        const val FORCED_ROLLBACK_MESSAGE = "integration-forced-rollback-after-agents"

        // Исключаем только технические поля: даты создания и обновления
        // могут намеренно меняться при повторном импорте.
        val BUSINESS_TABLES = listOf(
            "ai_agent", "applied_metric", "agent_contact", "contact",
            "implemented_platform", "involved_resource", "agent_status_sla",
            "agent_quality_gate", "agent_process", "agent_enabler",
            "platform", "program", "process", "block", "division"
        )
    }

    @BeforeEach
    fun prepareTest() {
        createdUploadIds.clear()
        asyncUploadIds.clear()
        mockCurrentDate(LocalDate.of(2025, 1, 1))

        every { internalTerBanksApiClient.findAll() } returns ResponseEntity.ok(
            listOf(TerBank().apply {
                id = 12
                code = 0
                name = "Центральный аппарат"
                shortName = "ЦА"
            })
        )

        // Используем integration-block: этот код уже есть в предоставленном
        // integration/clearTables.sql. В отличие от aib-integration-block,
        // он будет удалён существующей очисткой.
        seedBlockIfMissing(TEST_BLOCK_CODE, TEST_BLOCK_NAME, "Корпоративно-инвестиционный бизнес")
        jdbcTemplate.update(
            "UPDATE upload_permission SET upload_allowed = true, last_updated = clock_timestamp() WHERE id = 'GLOBAL'"
        )
    }

    @AfterEach
    fun cleanup() {
        // Нельзя TRUNCATE бизнес-таблицы, пока продолжается async worker.
        asyncUploadIds.forEach { waitUntilFinishedAndUnlocked(it) }
        removeLateFailureTrigger()

        executeSqlScript("integration/sql/clearTables.sql")

        // Существующий скрипт не очищает file_upload. Удаляем только записи
        // конкретно этого теста, не затрагивая соседние тесты.
        createdUploadIds.forEach { uploadId ->
            if (fileUploadRepository.existsById(uploadId)) {
                fileUploadRepository.deleteById(uploadId)
            }
        }
        jdbcTemplate.update(
            "UPDATE upload_permission SET upload_allowed = true, last_updated = clock_timestamp() WHERE id = 'GLOBAL'"
        )
    }

    private fun seedBlockIfMissing(code: String, shortName: String, name: String) {
        val count = jdbcTemplate.queryForObject(
            "SELECT count(*) FROM block WHERE code = ?", Int::class.java, code
        ) ?: 0
        if (count == 0) {
            jdbcTemplate.update(
                "INSERT INTO block (code, short_name, name, disabled) VALUES (?, ?, ?, false)",
                code, shortName, name
            )
        }
    }

    private fun xlsx(): MockMultipartFile = multipart(Files.readAllBytes(Paths.get(FIXTURE_PATH)))

    private fun multipart(bytes: ByteArray): MockMultipartFile = MockMultipartFile(
        "file", "agents.xlsx",
        "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet", bytes
    )

    private fun startImport(file: MockMultipartFile = xlsx()): Long {
        val fileUploadId = excelUploadService.upload(file, TEST_USER_ID)
        createdUploadIds.add(fileUploadId)
        asyncUploadIds.add(fileUploadId)
        return fileUploadId
    }

    /**
     * SUCCESS/ERROR в БД может появиться раньше, чем сработает finally worker.
     * Поэтому возвращаемся только когда status терминальный И GLOBAL освобождён.
     */
    private fun waitUntilFinishedAndUnlocked(fileUploadId: Long): FileUploadEntity {
        await().atMost(120, TimeUnit.SECONDS).untilAsserted {
            val status = fileUploadRepository.findById(fileUploadId).orElseThrow().status
            assertTrue(
                status == FileUploadStatus.SUCCESS || status == FileUploadStatus.ERROR,
                "Импорт $fileUploadId ещё выполняется: status=$status"
            )
            assertTrue(isUploadAllowed(), "Импорт $fileUploadId завершился, но GLOBAL ещё заблокирован")
        }
        return fileUploadRepository.findById(fileUploadId).orElseThrow()
    }

    private fun isUploadAllowed(): Boolean = jdbcTemplate.queryForObject(
        "SELECT upload_allowed FROM upload_permission WHERE id = 'GLOBAL'", Boolean::class.java
    ) == true

    /**
     * Снимок бизнес-значений для проверки повторного импорта и rollback.
     * PK agent_contact/applied_metric включены: неизменённые строки теперь
     * должны сохранять идентификаторы, а не физически пересоздаваться.
     * Исключены только временные и аудиторские поля, зависящие от момента загрузки.
     */
    private fun businessSnapshot(): Map<String, List<String>> = BUSINESS_TABLES.associateWith { table ->
        jdbcTemplate.queryForList(
            """
                SELECT (to_jsonb(t) - 'created' - 'updated' - 'created_by' - 'updated_by')::text
                FROM "$table" t
                ORDER BY 1
            """.trimIndent(),
            String::class.java
        )
    }

    @Test
    fun `should import same Excel twice without changing business data or increasing counts`() {
        val first = waitUntilFinishedAndUnlocked(startImport())
        assertEquals(FileUploadStatus.SUCCESS, first.status, "Первая загрузка: ${first.errorDescription}")

        val firstSnapshot = businessSnapshot()
        val agentCount = aiAgentRepository.count()
        val metricCount = appliedMetricRepository.count()
        assertTrue(agentCount > 0, "Фикстура не создала агентов: проверка была бы бесполезна")

        val second = waitUntilFinishedAndUnlocked(startImport())
        assertEquals(FileUploadStatus.SUCCESS, second.status, "Повторная загрузка: ${second.errorDescription}")
        assertTrue(isUploadAllowed())
        assertEquals(agentCount, aiAgentRepository.count())
        assertEquals(metricCount, appliedMetricRepository.count())
        assertEquals(firstSnapshot, businessSnapshot(), "Повторный импорт изменил бизнес-данные")
    }

    @Test
    fun `should fill legacy success audit fields`() {
        val uploaded = waitUntilFinishedAndUnlocked(startImport())
        assertEquals(FileUploadStatus.SUCCESS, uploaded.status)
        assertNull(uploaded.errorDescription, "SUCCESS не должен оставлять Processing started...")
        assertEquals(TEST_USER_ID, uploaded.updatedBy)
        assertNotNull(uploaded.uploaded)
        assertNotNull(uploaded.updated)
    }

    @Test
    fun `should preserve unchanged metric ids and audit timestamps on repeated import`() {
        val first = waitUntilFinishedAndUnlocked(startImport())
        assertEquals(FileUploadStatus.SUCCESS, first.status)

        val before = jdbcTemplate.queryForList(
            "SELECT id, ai_agent_id, metric_id, type, current_value, created, updated FROM applied_metric ORDER BY id"
        )
        assertTrue(before.isNotEmpty(), "Для этого теста XLSX должен содержать хотя бы одну метрику")

        val second = waitUntilFinishedAndUnlocked(startImport())
        assertEquals(FileUploadStatus.SUCCESS, second.status)

        val after = jdbcTemplate.queryForList(
            "SELECT id, ai_agent_id, metric_id, type, current_value, created, updated FROM applied_metric ORDER BY id"
        )
        assertEquals(before, after, "Неизменённые метрики не должны пересоздаваться или обновляться")
    }

    @Test
    fun `should update changed metric value without replacing its row`() {
        val first = waitUntilFinishedAndUnlocked(startImport())
        assertEquals(FileUploadStatus.SUCCESS, first.status)

        val before = jdbcTemplate.queryForMap(
            "SELECT id, current_value, created FROM applied_metric ORDER BY id LIMIT 1"
        )
        val metricId = (before.getValue("id") as Number).toLong()
        val originalValue = before.getValue("current_value") as java.math.BigDecimal
        val created = before["created"]

        jdbcTemplate.update(
            "UPDATE applied_metric SET current_value = ? WHERE id = ?",
            originalValue.add(java.math.BigDecimal("123.0")), metricId
        )

        val second = waitUntilFinishedAndUnlocked(startImport())
        assertEquals(FileUploadStatus.SUCCESS, second.status)
        val after = jdbcTemplate.queryForMap(
            "SELECT id, current_value, created FROM applied_metric WHERE id = ?", metricId
        )
        assertEquals(metricId, (after.getValue("id") as Number).toLong())
        assertEquals(0, (after.getValue("current_value") as java.math.BigDecimal).compareTo(originalValue))
        assertEquals(created, after["created"], "Изменение значения не должно заменять метрику")
    }

    @Test
    fun `should preserve imported contact row but clear legacy non PULT user ownership`() {
        val first = waitUntilFinishedAndUnlocked(startImport())
        assertEquals(FileUploadStatus.SUCCESS, first.status)

        // Ищем импортированную ссылку из настоящего XLSX, а не синтетическую:
        // после повторного импорта ссылка должна иметь тот же id.
        val contactLink = jdbcTemplate.queryForList(
            """
                SELECT ac.id, ac.agent_id, ac.contact_id, ac.type
                FROM agent_contact ac
                JOIN ai_agent agent ON agent.id = ac.agent_id
                WHERE agent.agent_id NOT LIKE '%PULT%'
                  AND agent.import_status IS DISTINCT FROM 'blocked'
                ORDER BY ac.id
                LIMIT 1
            """.trimIndent()
        ).firstOrNull() ?: error("Для теста нужен non-PULT контакт из XLSX")

        val linkId = (contactLink.getValue("id") as Number).toLong()
        val expectedAgentId = (contactLink.getValue("agent_id") as Number).toLong()
        val expectedContactId = (contactLink.getValue("contact_id") as Number).toLong()
        val expectedType = contactLink["type"]
        jdbcTemplate.update("UPDATE agent_contact SET user_id = ? WHERE id = ?", 987654321L, linkId)

        val second = waitUntilFinishedAndUnlocked(startImport())
        assertEquals(FileUploadStatus.SUCCESS, second.status)

        val after = jdbcTemplate.queryForMap(
            "SELECT id, agent_id, contact_id, type, user_id FROM agent_contact WHERE id = ?", linkId
        )
        assertEquals(linkId, (after.getValue("id") as Number).toLong())
        assertEquals(expectedAgentId, (after.getValue("agent_id") as Number).toLong())
        assertEquals(expectedContactId, (after.getValue("contact_id") as Number).toLong())
        assertEquals(expectedType, after["type"])
        assertNull(after["user_id"], "У non-PULT импортной связи userId должен стать null")
    }

    @Test
    fun `should remove obsolete non PULT contact links but retain unchanged ones`() {
        val first = waitUntilFinishedAndUnlocked(startImport())
        assertEquals(FileUploadStatus.SUCCESS, first.status)

        val contactLink = jdbcTemplate.queryForList(
            """
                SELECT ac.id, ac.agent_id, ac.contact_id
                FROM agent_contact ac
                JOIN ai_agent agent ON agent.id = ac.agent_id
                WHERE agent.agent_id NOT LIKE '%PULT%'
                  AND agent.import_status IS DISTINCT FROM 'blocked'
                ORDER BY ac.id LIMIT 1
            """.trimIndent()
        ).firstOrNull() ?: error("Для теста нужен non-PULT контакт из XLSX")
        val existingLinkId = (contactLink.getValue("id") as Number).toLong()
        val agentId = (contactLink.getValue("agent_id") as Number).toLong()
        val contactId = (contactLink.getValue("contact_id") as Number).toLong()
        val obsoleteId = jdbcTemplate.queryForObject(
            """
                INSERT INTO agent_contact (agent_id, contact_id, type, user_id)
                VALUES (?, ?, 'obsolete-integration-contact', 987654321)
                RETURNING id
            """.trimIndent(), Long::class.java, agentId, contactId
        ) ?: error("Не удалось создать устаревшую тестовую ссылку")

        val second = waitUntilFinishedAndUnlocked(startImport())
        assertEquals(FileUploadStatus.SUCCESS, second.status)
        assertTrue(jdbcTemplate.queryForObject(
            "SELECT EXISTS(SELECT 1 FROM agent_contact WHERE id = ?)", Boolean::class.java, existingLinkId
        ) == true, "Неизменённая ссылка была пересоздана")
        assertTrue(jdbcTemplate.queryForObject(
            "SELECT EXISTS(SELECT 1 FROM agent_contact WHERE id = ?)", Boolean::class.java, obsoleteId
        ) == false, "Устаревшая non-PULT ссылка не удалена")
    }

    @Test
    fun `should update agent name from modified Excel and keep metrics unchanged`() {
        val initial = waitUntilFinishedAndUnlocked(startImport())
        assertEquals(FileUploadStatus.SUCCESS, initial.status, initial.errorDescription)
        val metricsBefore = businessSnapshot().getValue("applied_metric")
        val countBefore = aiAgentRepository.count()

        val (modifiedFile, changedAgentId, changedName) = xlsxWithChangedAgentName()
        val updated = waitUntilFinishedAndUnlocked(startImport(modifiedFile))
        assertEquals(FileUploadStatus.SUCCESS, updated.status, updated.errorDescription)

        assertEquals(changedName, aiAgentRepository.findFirstByAgentId(changedAgentId)?.agentName)
        assertEquals(countBefore, aiAgentRepository.count())
        assertEquals(metricsBefore, businessSnapshot().getValue("applied_metric"))
    }

    private fun xlsxWithChangedAgentName(): Triple<MockMultipartFile, String, String> {
        XSSFWorkbook(ByteArrayInputStream(Files.readAllBytes(Paths.get(FIXTURE_PATH)))).use { workbook ->
            val sheet = workbook.getSheet(AGENTS_SHEET_NAME)
            val header = sheet.getRow(1) // В AgentSheetService заголовки — вторая строка.
            val formatter = DataFormatter()
            val idColumn = (0 until header.lastCellNum.toInt()).first { index ->
                formatter.formatCellValue(header.getCell(index)) == AGENT_ID
            }
            val nameColumn = (0 until header.lastCellNum.toInt()).first { index ->
                formatter.formatCellValue(header.getCell(index)) == AGENT_NAME
            }
            val agentRow = (2..sheet.lastRowNum)
                .mapNotNull(sheet::getRow)
                .first { row -> formatter.formatCellValue(row.getCell(idColumn)).isNotBlank() }

            val agentId = formatter.formatCellValue(agentRow.getCell(idColumn)).trim()
            val originalName = aiAgentRepository.findFirstByAgentId(agentId)?.agentName
            assertNotNull(originalName, "Не найден агент из первой загрузки: $agentId")
            val changedName = "$originalName - integration updated"
            assertNotEquals(originalName, changedName)

            agentRow.getCell(nameColumn, Row.MissingCellPolicy.CREATE_NULL_AS_BLANK)
                .setCellValue(changedName)
            val bytes = ByteArrayOutputStream().use { output ->
                workbook.write(output)
                output.toByteArray()
            }
            return Triple(multipart(bytes), agentId, changedName)
        }
    }

    /**
     * Ошибка на завершающем UPDATE file_upload.status=SUCCESS, когда все
     * бизнес-изменения уже выполнены в этой же транзакции.
     * PostgreSQL trigger создаётся только в embedded БД теста.
     * Никаких production-миграций или дополнительных таблиц.
     */
    private fun installLateFailureTrigger() {
        jdbcTemplate.execute(
            """
                CREATE OR REPLACE FUNCTION integration_fail_excel_import_success()
                RETURNS trigger AS ${'$'}${'$'}
                BEGIN
                    RAISE EXCEPTION '$FORCED_ROLLBACK_MESSAGE';
                END;
                ${'$'}${'$'} LANGUAGE plpgsql
            """.trimIndent()
        )
        jdbcTemplate.execute(
            """
                CREATE TRIGGER integration_fail_excel_import_success
                BEFORE UPDATE OF status ON file_upload
                FOR EACH ROW WHEN (lower(NEW.status::text) = 'success')
                EXECUTE FUNCTION integration_fail_excel_import_success()
            """.trimIndent()
        )
    }

    private fun removeLateFailureTrigger() {
        jdbcTemplate.execute("DROP TRIGGER IF EXISTS integration_fail_excel_import_success ON file_upload")
        jdbcTemplate.execute("DROP FUNCTION IF EXISTS integration_fail_excel_import_success()")
    }

    @Test
    fun `should roll back all business tables when final success update fails`() {
        val before = businessSnapshot()
        val agentCountBefore = aiAgentRepository.count()
        installLateFailureTrigger()

        val failed = waitUntilFinishedAndUnlocked(startImport())
        assertEquals(FileUploadStatus.ERROR, failed.status)
        assertTrue(
            failed.errorDescription?.contains(FORCED_ROLLBACK_MESSAGE) == true,
            "Ошибка должна быть от тестового PostgreSQL trigger: ${failed.errorDescription}"
        )
        assertEquals(before, businessSnapshot(), "После ERROR остались частично применённые данные")
        assertEquals(agentCountBefore, aiAgentRepository.count())
        assertTrue(isUploadAllowed(), "После rollback не освободилась блокировка")
    }

    @Test
    fun `should roll back updates to existing agents when final success update fails`() {
        val initial = waitUntilFinishedAndUnlocked(startImport())
        assertEquals(FileUploadStatus.SUCCESS, initial.status, initial.errorDescription)
        val before = businessSnapshot()
        val (modifiedFile, changedAgentId, changedName) = xlsxWithChangedAgentName()
        installLateFailureTrigger()

        val failed = waitUntilFinishedAndUnlocked(startImport(modifiedFile))
        assertEquals(FileUploadStatus.ERROR, failed.status)
        assertTrue(
            failed.errorDescription?.contains(FORCED_ROLLBACK_MESSAGE) == true,
            "Ошибка должна быть от тестового PostgreSQL trigger: ${failed.errorDescription}"
        )
        assertEquals(before, businessSnapshot(), "Изменения существующих данных не откатились")
        assertNotEquals(changedName, aiAgentRepository.findFirstByAgentId(changedAgentId)?.agentName)
    }

    @Test
    fun `should leave business tables unchanged when preparation fails`() {
        val before = businessSnapshot()
        every { internalTerBanksApiClient.findAll() } throws
                IllegalStateException("integration-terbanks-unavailable")

        val failed = waitUntilFinishedAndUnlocked(startImport())
        assertEquals(FileUploadStatus.ERROR, failed.status)
        assertEquals(before, businessSnapshot())
        assertTrue(isUploadAllowed())
    }

    @Test
    fun `should not let stale registration release lock owned by another import`() {
        val first = registrationService.register("first.xlsx", TEST_USER_ID)
        createdUploadIds.add(first.fileUploadId)
        var second: RegisteredExcelImport? = null
        try {
            // Имитируем административную разблокировку ПОСЛЕ гибели старого worker.
            // Реальный worker в этом тесте не запускался.
            jdbcTemplate.update(
                "UPDATE upload_permission SET upload_allowed = true, last_updated = clock_timestamp() WHERE id = 'GLOBAL'"
            )
            second = registrationService.register("second.xlsx", TEST_USER_ID)
            createdUploadIds.add(second.fileUploadId)

            permissionService.release(first.acquiredAt, first.userId)
            assertTrue(!isUploadAllowed(), "Старый владелец снял чужую блокировку")
        } finally {
            second?.let { permissionService.release(it.acquiredAt, it.userId) }
        }
        assertTrue(isUploadAllowed())
    }

    @Test
    fun `should allow only one of two concurrent registrations`() {
        val executor = Executors.newFixedThreadPool(2)
        val ready = CountDownLatch(2)
        val start = CountDownLatch(1)
        val register = Callable<RegisteredExcelImport?> {
            ready.countDown()
            check(start.await(10, TimeUnit.SECONDS)) { "Registration start timeout" }
            try {
                registrationService.register("parallel.xlsx", TEST_USER_ID)
            } catch (_: AiResponseException) {
                null
            }
        }

        val firstFuture = executor.submit(register)
        val secondFuture = executor.submit(register)
        val acquired = mutableListOf<RegisteredExcelImport>()
        try {
            assertTrue(ready.await(10, TimeUnit.SECONDS), "Не стартовали оба потока")
            start.countDown()
            val results = listOf(
                firstFuture.get(20, TimeUnit.SECONDS),
                secondFuture.get(20, TimeUnit.SECONDS)
            )
            acquired.addAll(results.filterNotNull())
            createdUploadIds.addAll(acquired.map { it.fileUploadId })
            assertEquals(1, acquired.size, "Два потока одновременно получили GLOBAL")
            assertTrue(!isUploadAllowed(), "При захваченном GLOBAL должен быть false")
        } finally {
            start.countDown()
            acquired.forEach { registration ->
                permissionService.release(registration.acquiredAt, registration.userId)
            }
            executor.shutdownNow()
        }
        assertTrue(isUploadAllowed(), "GLOBAL не освободился после release")
    }
}
```
