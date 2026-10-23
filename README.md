```java
data class PreparedCell(
    val columnIndex: Int,
    val rowIndex: Int,
    val cellType: CellType,
    val cachedFormulaResultType: CellType,
    private val textValue: String?,
    private val numberValue: Double?,
    private val booleanValue: Boolean?,
    private val dateMillis: Long?,
    val isDateFormatted: Boolean,
    val formattedValue: String? = null
) : Serializable {

    val stringCellValue: String
        get() = textValue
            ?: throw IllegalStateException("Non-string Excel cell at row=$rowIndex, column=$columnIndex")

    val numericCellValue: Double
        get() = numberValue
            ?: throw IllegalStateException("Non-numeric Excel cell at row=$rowIndex, column=$columnIndex")

    val booleanCellValue: Boolean
        get() = booleanValue
            ?: throw IllegalStateException("Non-boolean Excel cell at row=$rowIndex, column=$columnIndex")

    val dateCellValue: Date?
        get() = dateMillis?.let(::Date)

    companion object {
        fun capture(cell: Cell, formatter: DataFormatter): PreparedCell {
            val effectiveType = if (cell.cellType == CellType.FORMULA) cell.cachedFormulaResultType else cell.cellType
            val formattedDate = (effectiveType == CellType.NUMERIC && DateUtil.isCellDateFormatted(cell))
            return PreparedCell(
                columnIndex = cell.columnIndex,
                rowIndex = cell.rowIndex,
                cellType = cell.cellType,
                cachedFormulaResultType = if (cell.cellType == CellType.FORMULA) cell.cachedFormulaResultType else cell.cellType,
                textValue = when (effectiveType) {
                    CellType.STRING -> cell.stringCellValue
                    CellType.BLANK -> ""
                    else -> null
                },
                numberValue = if (effectiveType == CellType.NUMERIC) cell.numericCellValue else null,
                booleanValue = if (effectiveType == CellType.BOOLEAN) cell.booleanCellValue else null,
                dateMillis = if (formattedDate) cell.dateCellValue?.time else null,
                isDateFormatted = formattedDate,
                formattedValue = if (effectiveType == CellType.NUMERIC) {
                    if (cell.cellType == CellType.FORMULA) {
                        // У formula-cells DataFormatter.formatCellValue без evaluator
                        // может вернуть текст формулы, а не кэшированное число.
                        val cellStyle = cell.cellStyle
                        formatter.formatRawCellContents(
                            cell.numericCellValue,
                            cellStyle.dataFormat.toInt(),
                            cellStyle.dataFormatString ?: "General"
                        )
                    } else {
                        formatter.formatCellValue(cell)
                    }
                } else null
            )
        }
    }
}

/**
 * Не содержит Hibernate/JPA-сущностей; подготовленную строку можно читать многократно.
 */
class PreparedRow(val rowNum: Int, private val cells: List<PreparedCell>) : Iterable<PreparedCell>, Serializable {
    @Transient private var indexedCells: Map<Int, PreparedCell>? = null

    fun getCell(index: Int): PreparedCell? {
        val indexByColumn = indexedCells ?: cells.associateBy(PreparedCell::columnIndex).also { indexedCells = it }
        return indexByColumn[index]
    }

    override fun iterator(): Iterator<PreparedCell> = cells.iterator()
}

/**
 * Бинарная временная таблица строк одного листа на диске; не является таблицей PostgreSQL.
 */
class PreparedSheet(private val path: Path, val rowCount: Int) {
    fun <T> readRows(block: (Iterator<PreparedRow>) -> T): T {
        ObjectInputStream(BufferedInputStream(Files.newInputStream(path))).use { input ->
            var remaining = rowCount
            val iterator = object : Iterator<PreparedRow> {
                override fun hasNext(): Boolean = remaining > 0
                override fun next(): PreparedRow {
                    if (!hasNext()) throw NoSuchElementException()
                    remaining--
                    return input.readObject() as PreparedRow
                }
            }
            return block(iterator)
        }
    }
}

```

## PreparedExcelWorkbook.kt

```kotlin
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
            val formatter = DataFormatter(Locale.ROOT)
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
                                        PreparedCell.capture(cell, formatter)
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

```

## DictAndProcessService.kt

```kotlin
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

    /**
     * Чтение текстовых реквизитов DICT и PROCESSES с сохранением формата Excel.
     * Строковые коды (например, "0321") не преобразуются в числа.
     * Числовые ячейки с форматом 0000 читаются как отображаемое значение "0321".
     * Ячейки с настоящей датой и прочие неподдерживаемые типы отклоняются.
     */
    private fun getDictCellValue(cell: PreparedCell?): String {
        if (cell == null) return ""

        val effectiveType = if (cell.cellType == CellType.FORMULA) {
            cell.cachedFormulaResultType
        } else {
            cell.cellType
        }

        return when (effectiveType) {
            CellType.STRING -> cell.stringCellValue
            CellType.BLANK -> ""
            CellType.NUMERIC -> {
                if (cell.isDateFormatted) {
                    throw IllegalArgumentException(
                        "Unexpected date in dictionary at row=${cell.rowIndex + 1}, column=${cell.columnIndex + 1}"
                    )
                }
                cell.formattedValue
                    ?: BigDecimal.valueOf(cell.numericCellValue).stripTrailingZeros().toPlainString()
            }
            else -> throw IllegalArgumentException(
                "Unsupported dictionary cell type: $effectiveType at row=${cell.rowIndex + 1}, column=${cell.columnIndex + 1}"
            )
        }
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
        val tribeLabel = getDictCellValue(row.getCell(tribeLabelCol))
        val shortName = getDictCellValue(row.getCell(tribeShortNameCol))
        val tribeCode = getDictCellValue(row.getCell(tribeCodeCol))
        val tribeBlock = getDictCellValue(row.getCell(tribeBlockCol))
        val tribeName = getDictCellValue(row.getCell(tribeNameCol))
        if (tribeLabel.isNotBlank() || shortName.isNotBlank() || tribeCode.isNotBlank() || tribeBlock.isNotBlank() || tribeName.isNotBlank()) {
            divisions.add(DivisionsDto(tribeLabel, shortName, tribeCode, tribeBlock, tribeName))
        }

        val surfaceName = getDictCellValue(row.getCell(surfaceNameCol))
        val surfaceCode = getDictCellValue(row.getCell(surfaceCodeCol))
        if (surfaceName.isNotBlank() || surfaceCode.isNotBlank()) {
            platforms.add(PlatformsDto(surfaceName, surfaceCode))
        }

        val blockShortName = getDictCellValue(row.getCell(blockShortNameCol))
        val blockCode = getDictCellValue(row.getCell(blockCodeCol))
        val blockName = getDictCellValue(row.getCell(blockNameCol))
        val blockCurator = getDictCellValue(row.getCell(blockCuratorNameCol))
        if (blockShortName.isNotBlank() || blockCode.isNotBlank() || blockName.isNotBlank() || blockCurator.isNotBlank()) {
            blocks.add(BlocksDto(blockCode, blockShortName, blockName, blockCurator))
        }

        val programFileName = getDictCellValue(row.getCell(programFileNameCol))
        val programCode = getDictCellValue(row.getCell(programCodeCol))
        val programName = getDictCellValue(row.getCell(programNameCol))
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
                    val processShortName = getDictCellValue(row.getCell(processShortNameCol))
                    val processProgram = getDictCellValue(row.getCell(processProgramCol))
                    val processBlock = getDictCellValue(row.getCell(processBlockCol))
                    val processName = getDictCellValue(row.getCell(processNameCol))

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

                    val processMappingDpss = getDictCellValue(row.getCell(processMappingDpssCol))
                    val processMappingAris = getDictCellValue(row.getCell(processMappingArisCol))
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

```

## PreparedCellFormattingTest.kt

```kotlin
import org.apache.poi.ss.usermodel.DataFormatter
import org.apache.poi.xssf.usermodel.XSSFWorkbook
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Test
import java.util.Locale

class PreparedCellFormattingTest {
    private val formatter = DataFormatter(Locale.ROOT)

    @Test
    fun `text code keeps leading zero`() {
        XSSFWorkbook().use { workbook ->
            val cell = workbook.createSheet().createRow(0).createCell(0)
            cell.setCellValue("0321")
            val prepared = PreparedCell.capture(cell, formatter)
            assertEquals("0321", prepared.stringCellValue)
        }
    }

    @Test
    fun `numeric code with 0000 format keeps leading zero`() {
        XSSFWorkbook().use { workbook ->
            val cell = workbook.createSheet().createRow(0).createCell(0)
            val style = workbook.createCellStyle()
            style.dataFormat = workbook.createDataFormat().getFormat("0000")
            cell.cellStyle = style
            cell.setCellValue(321.0)
            val prepared = PreparedCell.capture(cell, formatter)
            assertEquals("0321", prepared.formattedValue)
        }
    }

    @Test
    fun `general numeric code stays unpadded`() {
        XSSFWorkbook().use { workbook ->
            val cell = workbook.createSheet().createRow(0).createCell(0)
            cell.setCellValue(321.0)
            val prepared = PreparedCell.capture(cell, formatter)
            assertEquals("321", prepared.formattedValue)
        }
    }
}



```
