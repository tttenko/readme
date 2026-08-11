```java
@Suppress("UNUSED_EXPRESSION")
@Service
class AgentSheetService(
    private val aIAgentRepository: AIAgentRepository,
    private val agentStatusSlaRepository: AgentStatusSlaRepository,
    private val appliedMetricRepository: AppliedMetricRepository,
    private val contactRepository: ContactRepository,
    private val processRepository: ProcessRepository,
    private val fileUploadRepository: FileUploadRepository,
    private val aiAgentQualityGateService: AiAgentQualityGateService,
    private val dateTimeProvider: DateTimeProvider,
    private val messageProvider: MessageProvider,
    private val transactionTemplate: TransactionTemplate,
    private val agentContactRepository: AgentContactRepository,
    private val entityManager: EntityManager,
    private val dictionariesProvider: AgentImportDictionariesProvider,
) {
    private val log by logger()
    private val emailRegex = Regex("^[a-zA-Z0-9._%+-]+@(sberbank\\.ru|sber\\.ru)$")


    private companion object {
        const val AGENT_BATCH_SIZE = 100
        const val BULK_BATCH_SIZE = 500
    }

    fun processAgentSheet(
        context: DictAndProcessContext,
        agentSheet: Sheet,
        userId: Long
    ) {
        val operationDetails = "Request 'process Agent page' with user=$userId "

        transactionTemplate.execute { status ->
            try {
                log.logBefore(operationDetails)

                /*
                 * Удаление старых связей контактов выполняется внутри той же транзакции,
                 * что и импорт. Если импорт упадет, удаление тоже откатится.
                 */
                deleteNotPULTAgents()

                /*
                 * Provider строит immutable snapshot справочников один раз на весь импорт.
                 * После этого persistence context очищается, а внутри batch используются
                 * только id/scalar snapshot и managed references текущей пачки.
                 */
                val dictionaries = dictionariesProvider.load()
                entityManager.clear()

                val sheetStructure = getSheetStructure(agentSheet)
                val agentsInFile = mutableSetOf<String>()
                val savedAgentIds = mutableListOf<Long>()
                val blockedAgents = mutableListOf<BlockedAgent>()

                val duration = measureTimeMillis {
                    val rowIterator = agentSheet.iterator()
                    val batch = ArrayList<Row>(AGENT_BATCH_SIZE)
                    var currentRowIndex = 0

                    while (rowIterator.hasNext()) {
                        val row = rowIterator.next()
                        currentRowIndex++

                        if (currentRowIndex <= DATA_START_ROW_INDEX) {
                            continue
                        }

                        batch.add(row)

                        if (batch.size == AGENT_BATCH_SIZE) {
                            savedAgentIds += processBatch(
                                rows = batch,
                                sheetStructure = sheetStructure,
                                dictionaries = dictionaries,
                                context = context,
                                agentsInFile = agentsInFile,
                                blockedAgents = blockedAgents,
                                userId = userId
                            )

                            /*
                             * flush отправляет SQL в PostgreSQL, но НЕ делает commit.
                             * Поэтому rollback всей загрузки по-прежнему возможен.
                             *
                             * clear освобождает first-level cache Hibernate и snapshots
                             * обработанных сущностей текущего batch.
                             */
                            entityManager.flush()
                            entityManager.clear()
                            batch.clear()
                        }
                    }

                    if (batch.isNotEmpty()) {
                        savedAgentIds += processBatch(
                            rows = batch,
                            sheetStructure = sheetStructure,
                            dictionaries = dictionaries,
                            context = context,
                            agentsInFile = agentsInFile,
                            blockedAgents = blockedAgents,
                            userId = userId
                        )
                        entityManager.flush()
                        entityManager.clear()
                        batch.clear()
                    }

                    val qualityGateDuration = measureTimeMillis {
                        savedAgentIds.chunked(BULK_BATCH_SIZE).forEach { ids ->
                            aiAgentQualityGateService.createOrUpdateByAgent(ids)
                            aiAgentQualityGateService.removeIrrelevant(ids)
                        }
                    }
                    log.debug("Operation QualityGateRefresh took $qualityGateDuration ms")
                }

                log.debug("Operation update entity took $duration ms")

                if (agentsInFile.isEmpty()) {
                    throw AiFileUploadException(
                        errorCode = AI_UPLOAD_EMPTY_FILE,
                        message = MessageFormat.format(messageProvider[AI_UPLOAD_EMPTY_FILE]),
                        fileId = context.fileUploadId
                    )
                }

                disableDeletedAgents(agentsInFile)

                /*
                 * Не загружаем process.agents целиком.
                 * Статусы процессов пересчитываются bulk-запросом в PostgreSQL.
                 */
                updateProcessStatus()

                /*
                 * После последнего clear() заново получаем только одну FileUploadEntity.
                 * Это гарантирует, что она managed и изменения blockedAgents/status
                 * попадут в текущую транзакцию.
                 */
                val fileUploadEntity = fileUploadRepository.findById(context.fileUploadId)
                    .orElseThrow {
                        IllegalStateException("FileUploadEntity not found: ${context.fileUploadId}")
                    }

                fileUploadEntity.blockedAgents = blockedAgents
                fileUploadEntity.status = SUCCESS
                fileUploadRepository.save(fileUploadEntity)

                entityManager.flush()
                log.logSuccess(operationDetails)
            } catch (exception: Exception) {
                status.setRollbackOnly()
                throw exception
            }
        }
    }


    private fun selectActualAgent(agents: List<AIAgentEntity>): AIAgentEntity {
        return agents.sortedWith(
            compareBy<AIAgentEntity> { it.disabled == true }
                .thenByDescending { it.updated ?: it.created ?: LocalDateTime.MIN }
                .thenByDescending { it.id ?: 0L }
        ).first()
    }



    private fun processBatch(
        rows: List<Row>,
        sheetStructure: SheetStructure,
        dictionaries: AgentImportDictionaries,
        context: DictAndProcessContext,
        agentsInFile: MutableSet<String>,
        blockedAgents: MutableList<BlockedAgent>,
        userId: Long
    ): List<Long> {
        val batchAgentIds = rows.mapNotNull { row ->
            sheetStructure.headers[AGENT_ID]
                ?.takeIf { it >= 0 }
                ?.let { index -> row.getCell(index)?.let(::getValueOrNull) }
                ?.takeIf { it.isNotBlank() }
        }.toSet()

        val existingAgents = if (batchAgentIds.isEmpty()) {
            emptyList()
        } else {
            aIAgentRepository.findAllByAgentIdIn(batchAgentIds)
        }

        val groupedAgentsByAgentId = existingAgents
            .filter { !it.agentId.isNullOrBlank() }
            .groupBy { it.agentId!! }

        val actualAgentsByAgentId = groupedAgentsByAgentId
            .mapValues { (_, agents) -> selectActualAgent(agents) }
            .toMutableMap()

        val duplicatedAgentsToDisable = groupedAgentsByAgentId
            .flatMap { (_, agents) ->
                val actualAgent = selectActualAgent(agents)
                agents.filter { it.id != actualAgent.id && it.disabled != true }
            }

        if (duplicatedAgentsToDisable.isNotEmpty()) {
            val now = LocalDateTime.now()
            duplicatedAgentsToDisable.forEach { duplicate ->
                duplicate.disabled = true
                duplicate.updated = now
            }
        }

        val actualAgentIds = actualAgentsByAgentId.values.mapNotNull { it.id }

        val slasByAgentId: MutableMap<Long, MutableList<AgentStatusSlaEntity>> =
            if (actualAgentIds.isEmpty()) {
                mutableMapOf()
            } else {
                agentStatusSlaRepository.findAllByAiAgentIdIn(actualAgentIds)
                    .filter { it.aiAgent?.id != null }
                    .groupBy { it.aiAgent!!.id!! }
                    .mapValues { (_, slas) -> slas.toMutableList() }
                    .toMutableMap()
            }

        val contactsByEmail = loadContactsForBatch(
            rows = rows,
            headers = sheetStructure.headers
        )

        val savedIds = ArrayList<Long>(rows.size)

        rows.forEach { row ->
            val oneEntityDuration = measureTimeMillis {
                processSingleRow(
                    row = row,
                    headers = sheetStructure.headers,
                    headerRow = sheetStructure.headerRow,
                    agentsInFile = agentsInFile,
                    dictionaries = dictionaries,
                    agentsByAgentId = actualAgentsByAgentId,
                    slasByAgentId = slasByAgentId,
                    contactsByEmail = contactsByEmail,
                    context = context,
                    fileId = context.fileUploadId,
                    blockedAgents = blockedAgents,
                    platformsStartIndex = sheetStructure.platformsStartIndex,
                    platformsEndIndex = sheetStructure.platformsEndIndex,
                    metricsStartIndex = sheetStructure.metricsStartIndex,
                    metricsEndIndex = sheetStructure.metricsEndIndex,
                    resourcesStartIndex = sheetStructure.resourcesStartIndex,
                    userId = userId
                )?.let(savedIds::add)
            }
            log.debug("Operation update one entity took $oneEntityDuration ms")
        }

        return savedIds
    }

    private fun loadContactsForBatch(
        rows: List<Row>,
        headers: Map<String, Int>
    ): MutableMap<String, ContactEntity> {
        val emails = rows.asSequence()
            .flatMap { row ->
                sequenceOf(CUSTOMER_CONTACT, DEVELOPER_CONTACT)
                    .mapNotNull { header ->
                        headers[header]
                            ?.let { row.getCell(it) }
                            ?.let(::getValueOrNull)
                            ?.let(::extractContactEmail)
                    }
            }
            .toSet()

        if (emails.isEmpty()) {
            return mutableMapOf()
        }

        return contactRepository.findAllByEmailIn(emails)
            .filter { !it.email.isNullOrBlank() }
            .associateByTo(mutableMapOf()) { it.email!! }
    }

    private fun extractContactEmail(contact: String): String? {
        if (contact == "0") return null

        val email = contact.split(',', ';').getOrNull(1) ?: return null
        if (!email.contains("@")) return null

        val pureEmail = email.removeSurrounding("<", ">").trim()
        return pureEmail.takeIf(emailRegex::matches)
    }


    private fun <T : Any> managedReference(type: Class<T>, id: Any): T =
        entityManager.getReference(type, id)

    private fun normalizeKey(value: String): String = value.trim().lowercase()


    private fun getSheetStructure(sheet: Sheet): SheetStructure {
        val iterator = sheet.iterator()
        var currentRowIndex = 0
        var headers = emptyMap<String, Int>()
        var platformsStartIndex = -1
        var platformsEndIndex = -1
        var metricsStartIndex = -1
        var metricsEndIndex = -1
        var resourcesStartIndex = -1
        var headerRow: Row? = null

        while (iterator.hasNext()) {
            val row = iterator.next()
            currentRowIndex++

            when (currentRowIndex) {
                1 -> {
                    for (cell in row) {
                        when (cell.stringCellValue) {
                            FRONTAL_CELL_INDEX -> platformsStartIndex = cell.columnIndex
                            RESOURCES_CELL_INDEX -> resourcesStartIndex = cell.columnIndex
                            METRICS_CELL_INDEX -> metricsStartIndex = cell.columnIndex
                        }
                    }
                    if (platformsStartIndex != -1 && resourcesStartIndex != -1) {
                        platformsEndIndex = resourcesStartIndex - 1
                    }
                    if (metricsStartIndex != -1 && platformsStartIndex != -1) {
                        metricsEndIndex = platformsStartIndex - 1
                    }
                }
                2 -> {
                    headerRow = row
                    headers = row.associate { cell ->
                        cell.stringCellValue to cell.columnIndex
                    }
                    break
                }
            }
        }

        return SheetStructure(
            headers = headers,
            headerRow = headerRow,
            platformsStartIndex = platformsStartIndex,
            platformsEndIndex = platformsEndIndex,
            metricsStartIndex = metricsStartIndex,
            metricsEndIndex = metricsEndIndex,
            resourcesStartIndex = resourcesStartIndex
        )
    }

    private data class SheetStructure(
        val headers: Map<String, Int>,
        val headerRow: Row?,
        val platformsStartIndex: Int,
        val platformsEndIndex: Int,
        val metricsStartIndex: Int,
        val metricsEndIndex: Int,
        val resourcesStartIndex: Int
    )

    private fun processSingleRow(
        row: Row,
        headers: Map<String, Int>,
        headerRow: Row?,
        agentsInFile: MutableSet<String>,
        dictionaries: AgentImportDictionaries,
        agentsByAgentId: MutableMap<String, AIAgentEntity>,
        slasByAgentId: MutableMap<Long, MutableList<AgentStatusSlaEntity>>,
        contactsByEmail: MutableMap<String, ContactEntity>,
        context: DictAndProcessContext,
        fileId: Long,
        blockedAgents: MutableList<BlockedAgent>,
        platformsStartIndex: Int,
        platformsEndIndex: Int,
        metricsStartIndex: Int,
        metricsEndIndex: Int,
        resourcesStartIndex: Int,
        userId: Long
    ): Long? {
        val agentId = headers[AGENT_ID]?.takeIf { it >= 0 }?.let { index ->
            row.getCell(index)?.let(::getValueOrNull)
        }

        if (agentId.isNullOrBlank()) return null

        checkDoublesAndAdd(agentId, agentsInFile, fileId = fileId)

        var newEntity = false

        var entity = agentsByAgentId[agentId]?.also { existing ->
            if (existing.importStatus == "blocked") {
                blockedAgents.add(
                    BlockedAgent(
                        agentId = existing.id.toString(),
                        agentName = existing.agentName ?: ""
                    )
                )
                return null
            }

            existing.agentStatus = updateAgentStatus(
                headers = headers,
                row = row,
                dictionaries = dictionaries,
                fileId = fileId
            )
            existing.updated = LocalDateTime.now()
        } ?: AIAgentEntity().also { created ->
            created.agentId = agentId
            created.agentStatus = updateAgentStatus(
                headers = headers,
                row = row,
                dictionaries = dictionaries,
                fileId = fileId
            )
            newEntity = true
        }

        updateAgentDivisionOrBlock(
            entity = entity,
            headers = headers,
            row = row,
            divisionsDict = context.divisionsDict,
            dictionaries = dictionaries,
            fileId = fileId
        )

        val entityUpdateDuration = measureTimeMillis {
            updateBaseEntityFields(
                entity = entity,
                headers = headers,
                row = row,
                dictionaries = dictionaries,
                processesMappingDtos = context.processData.second,
                contactsByEmail = contactsByEmail,
                fileId = fileId,
                userId = userId
            )
        }
        log.debug("Operation entity fields update took $entityUpdateDuration ms")

        /*
         * Для существующего агента save не нужен: он уже managed в текущем batch.
         * Новый агент сохраняем здесь, потому что его id нужен дочерним сущностям.
         */
        if (newEntity) {
            entity = aIAgentRepository.save(entity)
        }

        agentsByAgentId[agentId] = entity

        if (!newEntity) {
            appliedMetricRepository.deleteAllByAiAgentId(entity.id)
        }

        updateAppliedMetric(
            row = row,
            metricsStartIndex = metricsStartIndex,
            metricsEndIndex = metricsEndIndex,
            headerRow = headerRow,
            dictionaries = dictionaries,
            entity = entity,
            fileId = fileId
        )

        val agentSlas = slasByAgentId.getOrPut(entity.id!!) { mutableListOf() }

        val pocMvpTargetDecisionMetricDuration = measureTimeMillis {
            updateImplementationStatusesOfAgent(
                agent = entity,
                headers = headers,
                row = row,
                dictionaries = dictionaries,
                headerTypes = listOf(POC, MVP, TARGET_DECISION, CONFIRM_EFFECT),
                agentSlas = agentSlas
            )
        }
        log.debug("Operation pocMvpTargetDecision took $pocMvpTargetDecisionMetricDuration ms")

        updateInvolvedResource(
            row = row,
            headerRow = headerRow,
            resourcesStartIndex = resourcesStartIndex,
            dictionaries = dictionaries,
            entity = entity,
            fileId = fileId
        )

        updateAgentStatusSla(
            row = row,
            dictionaries = dictionaries,
            entity = entity,
            headers = headers,
            agentStatusesSla = agentSlas
        )

        if (context.terBanksRefs != null) {
            updateTerBanks(
                row = row,
                headers = headers,
                entity = entity,
                terBanksRefs = context.terBanksRefs,
                fileId = fileId
            )
        }

        updateImplementedPlatform(
            platformsStartIndex = platformsStartIndex,
            platformsEndIndex = platformsEndIndex,
            headerRow = headerRow,
            row = row,
            entity = entity,
            dictionaries = dictionaries
        )

        return entity.id
    }

    private fun updateAgentDivisionOrBlock(
        entity: AIAgentEntity,
        headers: Map<String, Int>,
        row: Row,
        divisionsDict: List<DivisionsDto>,
        dictionaries: AgentImportDictionaries,
        fileId: Long
    ) {
        val divisionCellValue = headers[DIVISION]?.let { headerIndex ->
            getValueOrNull(row.getCell(headerIndex))
                ?.takeUnless(String::isBlank)
        }

        if (divisionCellValue == null) {
            val blockCellValue = headers[BLOCK]?.let { headerIndex ->
                getValueOrNull(row.getCell(headerIndex))
                    ?.takeUnless(String::isBlank)
            }

            val blockRef = blockCellValue
                ?.let(::normalizeKey)
                ?.let(dictionaries.blocksByShortName::get)

            if (blockRef == null) {
                throw AiFileUploadException(
                    AI_UPLOAD_EMPTY_TRIBE_NAME,
                    messageProvider[AI_UPLOAD_EMPTY_TRIBE_NAME, blockCellValue, row.rowNum.plus(1)],
                    fileId = fileId
                )
            }

            entity.division = null
            entity.block = managedReference(BlockEntity::class.java, blockRef.id)
            return
        }

        val divisionRef = divisionsDict
            .firstOrNull { it.shortName.equals(divisionCellValue.trim(), ignoreCase = true) }
            ?.code
            ?.let(::normalizeKey)
            ?.let(dictionaries.divisionsByCode::get)

        if (divisionRef == null) {
            throw AiFileUploadException(
                AI_UPLOAD_EMPTY_TRIBE_NAME,
                messageProvider[AI_UPLOAD_EMPTY_TRIBE_NAME, row.rowNum.plus(1)],
                fileId = fileId
            )
        }

        entity.division = managedReference(DivisionEntity::class.java, divisionRef.id)
        entity.block = divisionRef.blockId?.let { blockId ->
            managedReference(BlockEntity::class.java, blockId)
        }
    }

    private fun updateProcessStatus() {
        processRepository.refreshStatusesFromAgents()
    }

    private fun updateAgentStatus(
        headers: Map<String, Int>,
        row: Row,
        dictionaries: AgentImportDictionaries,
        fileId: Long
    ): StatusEntity? {
        return headers[AGENT_STATUS]
            ?.let(row::getCell)
            ?.let { statusCell ->
                val statusValue = getValueOrNullFromFormula(statusCell)
                val statusRef = statusValue
                    ?.let(::normalizeKey)
                    ?.let(dictionaries.statusesByName::get)
                    ?: throw createException(
                        exceptionType = AI_UPLOAD_UNKNOWN_AGENT_STATUS,
                        row = row,
                        cell = statusCell,
                        fileId = fileId
                    )

                managedReference(StatusEntity::class.java, statusRef.id)
            }
    }

    private fun updateBaseEntityFields(
        entity: AIAgentEntity,
        headers: Map<String, Int>,
        row: Row,
        dictionaries: AgentImportDictionaries,
        processesMappingDtos: List<ProcessesMappingDto>,
        contactsByEmail: MutableMap<String, ContactEntity>,
        fileId: Long,
        userId: Long
    ) {
        entity.also { currentEntity ->
            currentEntity.processes =
                headers[PROCESS_CODE]?.let { columnIndex ->
                    row.getCell(columnIndex)?.let { cell ->
                        mapProcesses(
                            processesMappingDtos = processesMappingDtos,
                            processCodesFromCell = getProcessCodesFromCell(cell),
                            dictionaries = dictionaries,
                            rowNum = cell.rowIndex,
                            fileId = fileId
                        )
                    }
                } ?: mutableSetOf()

            currentEntity.program = headers[PROGRAM]?.let { columnIndex ->
                getValueOrNull(row.getCell(columnIndex))?.let { programFileName ->
                    val programRef = dictionaries.programsByFileName[normalizeKey(programFileName)]
                        ?: throw AiFileUploadException(
                            AI_UPLOAD_UNKNOWN_PROGRAM,
                            MessageFormat.format(
                                messageProvider[AI_UPLOAD_UNKNOWN_PROGRAM],
                                programFileName,
                                row.rowNum.plus(1)
                            ),
                            fileId = fileId
                        )

                    managedReference(ProgramEntity::class.java, programRef.id)
                }
            }

            currentEntity.agentName = headers[AGENT_NAME]?.let { columnIndex ->
                row.getCell(columnIndex)?.let(::getValueOrNull)
                    ?: throw AiFileUploadException(
                        AI_UPLOAD_AGENT_NAME_EMPTY,
                        MessageFormat.format(
                            messageProvider[AI_UPLOAD_AGENT_NAME_EMPTY],
                            row.rowNum.plus(1)
                        ),
                        fileId = fileId
                    )
            }

            currentEntity.agentDescription =
                headers[AGENT_DESCRIPTION]?.let { columnIndex ->
                    row.getCell(columnIndex)?.let(::getValueOrNull)?.take(1000)
                }

            currentEntity.agentProblem =
                headers[AGENT_PROBLEM]?.let { columnIndex ->
                    row.getCell(columnIndex)?.let(::getValueOrNull)?.take(1000)
                }

            currentEntity.agentSolution =
                headers[AGENT_SOLUTION]?.let { columnIndex ->
                    row.getCell(columnIndex)?.let(::getValueOrNull)?.take(1000)
                }

            currentEntity.agentJiraUrl =
                headers[AGENT_JIRA_URL]?.let { columnIndex ->
                    row.getCell(columnIndex)?.let(::getValueOrNull)
                }

            currentEntity.initiativeType =
                headers[AGENT_INITIATIVE_TYPE]?.let { columnIndex ->
                    row.getCell(columnIndex)?.let(::getValueOrNull)
                }?.let { fieldValue ->
                    val code = if (fieldValue.lowercase().contains("агент")) {
                        "agent"
                    } else {
                        "genAiSolution"
                    }

                    dictionaries.initiativeTypesByCode[code]?.let { ref ->
                        managedReference(InitiativeTypeEntity::class.java, ref.id)
                    }
                }

            val customerContact = headers[CUSTOMER_CONTACT]?.let { columnIndex ->
                row.getCell(columnIndex)?.let(::getValueOrNull)?.let { contactValue ->
                    createContact(
                        contact = contactValue,
                        contactsByEmail = contactsByEmail,
                        agent = currentEntity,
                        userId = userId,
                        type = "customer"
                    )
                }
            }
            customerContact?.let(currentEntity.agentContact::add)

            val developerContact = headers[DEVELOPER_CONTACT]?.let { columnIndex ->
                row.getCell(columnIndex)?.let(::getValueOrNull)?.let { contactValue ->
                    createContact(
                        contact = contactValue,
                        contactsByEmail = contactsByEmail,
                        agent = currentEntity,
                        userId = userId,
                        type = "developer"
                    )
                }
            }
            developerContact?.let(currentEntity.agentContact::add)

            currentEntity.agentEffectOptimization =
                headers[AGENT_EFFECT_OPTIMIZATION]?.let { columnIndex ->
                    row.getCell(columnIndex)?.let(::getValueOrNull)?.toDouble()?.toBigDecimal()
                }

            currentEntity.agentEffectRevenue =
                headers[AGENT_EFFECT_REVENUE]?.let { columnIndex ->
                    row.getCell(columnIndex)?.let(::getValueOrNull)?.toDouble()?.toBigDecimal()
                }

            currentEntity.disabled = false
        }
    }


    data class SlaData(
        val status: String,
        var dateValue: LocalDateTime? = null,
        var stringValue: String? = null
    )

    private fun updateImplementationStatusesOfAgent(
        agent: AIAgentEntity,
        headers: Map<String, Int>,
        row: Row,
        dictionaries: AgentImportDictionaries,
        headerTypes: List<String>,
        agentSlas: MutableList<AgentStatusSlaEntity>
    ) {
        val slas = headerTypes.mapNotNull { headerType ->
            headers[headerType]
                ?.let { columnIndex -> row.getCell(columnIndex) }
                ?.let { cell ->
                    SlaData(headerType).apply {
                        if (cell.cellType == CellType.NUMERIC) {
                            dateValue = extractDateTime(cell)
                        }
                        if (cell.cellType == CellType.STRING) {
                            stringValue = cell.stringCellValue
                        }
                    }
                }
        }.associateBy { it.status }

        slas.forEach { sla ->
            val statusCode = when (sla.key) {
                POC -> "analysis"
                MVP -> "development"
                TARGET_DECISION -> "pilot"
                CONFIRM_EFFECT -> "targetSolution"
                else -> null
            }

            val statusRef = statusCode
                ?.let(::normalizeKey)
                ?.let(dictionaries.statusesByCode::get)

            val agentSla = statusRef?.let { ref ->
                agentSlas.find { it.primaryKey.agentStatusId == ref.id }
            }

            if (sla.value.dateValue != null && statusRef != null) {
                val saved = agentStatusSlaRepository.save(
                    (agentSla ?: AgentStatusSlaEntity().apply {
                        agentStatus = managedReference(StatusEntity::class.java, statusRef.id)
                        aiAgent = agent
                    }).apply {
                        plannedDate = sla.value.dateValue
                    }
                )

                agentSlas.removeIf {
                    it.primaryKey.agentStatusId == saved.primaryKey.agentStatusId
                }
                agentSlas.add(saved)
            }

            if (
                sla.key == CONFIRM_EFFECT &&
                !slas[TARGET_DECISION]?.stringValue.equals("Завершен", ignoreCase = true)
            ) {
                agentSla?.apply {
                    completedDate = null
                }?.let { changed ->
                    val saved = agentStatusSlaRepository.save(changed)
                    agentSlas.removeIf {
                        it.primaryKey.agentStatusId == saved.primaryKey.agentStatusId
                    }
                    agentSlas.add(saved)
                }
            }

            if (
                slas[TARGET_DECISION]?.stringValue.equals("Завершен", ignoreCase = true) &&
                sla.key == CONFIRM_EFFECT &&
                statusRef != null
            ) {
                val saved = agentStatusSlaRepository.save(
                    (agentSla ?: AgentStatusSlaEntity().apply {
                        agentStatus = managedReference(StatusEntity::class.java, statusRef.id)
                        aiAgent = agent
                    }).apply {
                        completedDate = sla.value.dateValue ?: LocalDateTime.now()
                        plannedDate = this.plannedDate ?: sla.value.dateValue ?: LocalDateTime.now()
                    }
                )

                agentSlas.removeIf {
                    it.primaryKey.agentStatusId == saved.primaryKey.agentStatusId
                }
                agentSlas.add(saved)
            }
        }
    }


    private fun extractDateTime(cell: Cell?): LocalDateTime? {
        if (cell == null) return null

        return when (cell.cellType) {
            CellType.NUMERIC -> {
                if (DateUtil.isCellDateFormatted(cell)) {
                    cell.dateCellValue
                        ?.toInstant()
                        ?.atZone(ZoneId.systemDefault())
                        ?.toLocalDate()
                        ?.atStartOfDay()
                } else {
                    null
                }
            }

            CellType.FORMULA -> {
                if (cell.cachedFormulaResultType == CellType.NUMERIC && DateUtil.isCellDateFormatted(cell)) {
                    cell.dateCellValue
                        ?.toInstant()
                        ?.atZone(ZoneId.systemDefault())
                        ?.toLocalDate()
                        ?.atStartOfDay()
                } else {
                    null
                }
            }

            else -> null
        }
    }

    private fun updateImplementedPlatform(
        platformsStartIndex: Int,
        platformsEndIndex: Int,
        headerRow: Row?,
        row: Row,
        entity: AIAgentEntity,
        dictionaries: AgentImportDictionaries
    ) {
        val implementedPlatformDuration = measureTimeMillis {
            if (platformsStartIndex == -1 || platformsEndIndex == -1 || headerRow == null) return

            val entitiesToSave = row
                .filter { it.columnIndex in platformsStartIndex..platformsEndIndex }
                .mapNotNull { frontalCell ->
                    val cellHeader = headerRow.getCell(frontalCell.columnIndex)
                        ?.stringCellValue
                        ?.trim()
                        ?.takeIf { it.isNotBlank() }
                        ?: return@mapNotNull null

                    getValueOrNull(row.getCell(frontalCell.columnIndex))
                        ?.takeIf { it.isNotBlank() }
                        ?.let { rowFrontalValue ->
                            dictionaries.platformsByName[cellHeader]
                                ?.let { platformRef ->
                                    if (
                                        setOf("в разработке", "внедрен")
                                            .contains(rowFrontalValue.lowercase())
                                    ) {
                                        ImplementedPlatformEntity().apply {
                                            primaryKey = AIAgentPlatformPK().apply {
                                                aiAgentId = entity.id
                                                platformId = platformRef.id
                                            }
                                            platform = managedReference(
                                                PlatformEntity::class.java,
                                                platformRef.id
                                            )
                                            aiAgent = entity
                                            released = rowFrontalValue.equals(
                                                "Внедрен",
                                                ignoreCase = true
                                            )
                                        }
                                    } else {
                                        null
                                    }
                                }
                        }
                }

            entity.platforms.clear()
            entity.platforms.addAll(entitiesToSave)
        }

        log.debug("Operation implemented platforms took $implementedPlatformDuration ms")
    }


    private fun updateTerBanks(
        row: Row,
        headers: Map<String, Int>,
        entity: AIAgentEntity,
        terBanksRefs: MutableList<TerBank>,
        fileId: Long
    ) {
        headers[TER_BANK]?.let { headerIndex ->
            getCellValueAsString(row.getCell(headerIndex))?.let { terBankCellValue ->
                val terBank = terBanksRefs.find { it.shortName.lowercase() == terBankCellValue.lowercase() }
                if (terBank != null) {
                    entity.terbankId = terBank.id
                } else {
                    throw AiFileUploadException(
                        AI_UPLOAD_UNKNOWN_TER_BANK,
                        MessageFormat.format(
                            messageProvider[AI_UPLOAD_UNKNOWN_TER_BANK],
                            terBankCellValue,
                            row.rowNum.plus(1)
                        ),
                        fileId = fileId
                    )
                }
            }
        }
    }

    private fun updateAgentStatusSla(
        row: Row,
        dictionaries: AgentImportDictionaries,
        entity: AIAgentEntity,
        headers: Map<String, Int>,
        agentStatusesSla: List<AgentStatusSlaEntity>
    ) {
        val agentStatusSlaDuration = measureTimeMillis {
            headers[AGENT_STATUS]?.let { headerIndex ->
                var expirationDate: DeadlineExpiredType? = DeadlineExpiredType.ontime

                getValueOrNullFromFormula(row.getCell(headerIndex))?.let { status ->
                    val statusRef = dictionaries.statusesByName[normalizeKey(status)]

                    if (statusRef != null) {
                        val filteredStatusSlas = agentStatusesSla.filter { statusSla ->
                            val slaOrdering = statusSla.primaryKey.agentStatusId
                                ?.let(dictionaries.statusesById::get)
                                ?.ordering
                                ?: Long.MIN_VALUE

                            slaOrdering > (statusRef.ordering ?: Long.MIN_VALUE)
                        }

                        filteredStatusSlas.forEach { statusSla ->
                            statusSla.plannedDate?.let { plannedDate ->
                                if (
                                    plannedDate.toLocalDate().minusDays(14) <
                                    dateTimeProvider.currentDate()
                                ) {
                                    expirationDate = DeadlineExpiredType.expiration
                                }

                                if (
                                    plannedDate.toLocalDate() <
                                    dateTimeProvider.currentDate()
                                ) {
                                    expirationDate = DeadlineExpiredType.expired
                                }
                            }
                        }
                    } else if (
                        expirationDate != DeadlineExpiredType.expired &&
                        expirationDate != DeadlineExpiredType.expiration
                    ) {
                        expirationDate = DeadlineExpiredType.ontime
                    }

                    entity.deadlineExpired = expirationDate
                }
            }
        }

        log.debug("Operation agent status took $agentStatusSlaDuration ms")
    }

    private fun updateInvolvedResource(
        row: Row,
        headerRow: Row?,
        resourcesStartIndex: Int,
        dictionaries: AgentImportDictionaries,
        entity: AIAgentEntity,
        fileId: Long
    ) {
        val involvedResourceDuration = measureTimeMillis {
            if (resourcesStartIndex == -1 || headerRow == null) return

            val resourcesEndIndex = resourcesStartIndex + 3

            val entitiesToSave = row
                .filter { it.columnIndex in resourcesStartIndex..resourcesEndIndex }
                .mapNotNull { resource ->
                    val resourceName = headerRow.getCell(resource.columnIndex)
                        ?.stringCellValue
                        ?.trim()
                        ?.takeIf { it.isNotBlank() }
                        ?: return@mapNotNull null

                    getValueOrNull(row.getCell(resource.columnIndex))
                        ?.takeIf { it.isNotBlank() }
                        ?.let { rawValue ->
                            try {
                                val numericValue = rawValue.replace(',', '.').toBigDecimal()

                                dictionaries.resourcesByName[resourceName]
                                    ?.let { resourceEntity ->
                                        InvolvedResourceEntity().apply {
                                            id = InvolvedResourceEmbeddedId(
                                                aiAgentId = entity.id,
                                                source = resourceEntity.source,
                                                type = resourceEntity.type
                                            )
                                            value = numericValue
                                            aiAgent = entity
                                            updated = LocalDateTime.now()
                                        }
                                    }
                                    ?: throw AiFileUploadException(
                                        AI_UPLOAD_UNKNOWN_METRIC_NAME,
                                        MessageFormat.format(
                                            messageProvider[AI_UPLOAD_UNKNOWN_METRIC_NAME],
                                            resource.columnIndex.plus(1),
                                            row.rowNum.plus(1)
                                        ),
                                        fileId = fileId
                                    )
                            } catch (e: NumberFormatException) {
                                throw AiFileUploadException(
                                    AI_UPLOAD_UNKNOWN_METRIC_NAME,
                                    MessageFormat.format(
                                        messageProvider[AI_UPLOAD_UNKNOWN_METRIC_NAME],
                                        resource.columnIndex.plus(1),
                                        row.rowNum.plus(1)
                                    ),
                                    fileId = fileId
                                )
                            }
                        }
                }

            entity.involvedResource.clear()
            entity.involvedResource.addAll(entitiesToSave)
        }

        log.debug("Operation involved Resource took $involvedResourceDuration ms")
    }

    private fun updateAppliedMetric(
        row: Row,
        metricsStartIndex: Int,
        metricsEndIndex: Int,
        headerRow: Row?,
        dictionaries: AgentImportDictionaries,
        entity: AIAgentEntity,
        fileId: Long
    ) {
        val appliedMetricDuration = measureTimeMillis {
            if (metricsStartIndex == -1 || metricsEndIndex == -1 || headerRow == null) return

            val metricsToSave = mutableListOf<AppliedMetricsEntity>()

            row.filter { it.columnIndex in metricsStartIndex..metricsEndIndex }
                .forEach { metricValue ->
                    val metricName = headerRow.getCell(metricValue.columnIndex)
                        ?.stringCellValue
                        ?.trim()

                    if (metricName.equals("Индивидуальная метрика", ignoreCase = true)) {
                        return@forEach
                    }

                    getValueOrNull(row.getCell(metricValue.columnIndex))
                        ?.takeIf { it.isNotBlank() }
                        ?.let { rawValue ->
                            try {
                                val normalizedValue = rawValue.replace(',', '.').toDouble()

                                metricName
                                    ?.let(::normalizeKey)
                                    ?.let(dictionaries.metricsByFileName::get)
                                    ?.let { metricRef ->
                                        metricsToSave.add(
                                            AppliedMetricsEntity().also { appliedMetric ->
                                                appliedMetric.aiAgent = entity
                                                appliedMetric.metric = managedReference(
                                                    MetricEntity::class.java,
                                                    metricRef.id
                                                )
                                                appliedMetric.currentValue =
                                                    BigDecimal.valueOf(normalizedValue)
                                                appliedMetric.type = STANDARD
                                                appliedMetric.updated = LocalDateTime.now()
                                            }
                                        )
                                    }
                                    ?: throw AiFileUploadException(
                                        AI_UPLOAD_UNKNOWN_METRIC_NAME,
                                        MessageFormat.format(
                                            messageProvider[AI_UPLOAD_UNKNOWN_METRIC_NAME],
                                            metricValue.columnIndex.plus(1),
                                            row.rowNum.plus(1)
                                        ),
                                        fileId = fileId
                                    )
                            } catch (e: NumberFormatException) {
                                throw AiFileUploadException(
                                    AI_UPLOAD_UNKNOWN_METRIC_NAME,
                                    MessageFormat.format(
                                        messageProvider[AI_UPLOAD_UNKNOWN_METRIC_NAME],
                                        metricValue.columnIndex.plus(1),
                                        row.rowNum.plus(1)
                                    ),
                                    fileId = fileId
                                )
                            }
                        }
                }

            if (metricsToSave.isNotEmpty()) {
                appliedMetricRepository.saveAll(metricsToSave)
            }
        }

        log.debug("Operation applied Metric took $appliedMetricDuration ms")
    }

    private fun disableDeletedAgents(agentsInFile: Set<String>) {
        aIAgentRepository.findAllNonPultAgentRefs()
            .asSequence()
            .filter { it.agentId !in agentsInFile }
            .map { it.id }
            .chunked(BULK_BATCH_SIZE)
            .forEach { ids ->
                if (ids.isNotEmpty()) {
                    aIAgentRepository.disableByIds(ids)
                }
            }
    }

    private fun deleteNotPULTAgents() {
        agentContactRepository.deleteAllForImportedNonBlockedAgents()
    }

    private fun createContact(
        contact: String,
        contactsByEmail: MutableMap<String, ContactEntity>,
        agent: AIAgentEntity?,
        type: String,
        userId: Long
    ): AgentContactEntity? {
        val pureEmail = extractContactEmail(contact) ?: return null
        val fields = contact.split(',', ';')

        val contactEntity = contactsByEmail[pureEmail]?.apply {
            fio = fields.getOrNull(0)
        } ?: createAndSaveContactEntity(
            contactsByEmail = contactsByEmail,
            fields = fields,
            pureEmail = pureEmail
        )

        return AgentContactEntity(
            agent = agent,
            type = type,
            contact = contactEntity
        )
    }

    private fun createAndSaveContactEntity(
        contactsByEmail: MutableMap<String, ContactEntity>,
        fields: List<String>,
        pureEmail: String
    ): ContactEntity {
        val contactEntity = contactRepository.save(
            ContactEntity().apply {
                fio = fields.getOrNull(0)
                email = pureEmail
            }
        )

        contactsByEmail[pureEmail] = contactEntity
        return contactEntity
    }


    private fun getValueOrNull(cell: Cell): String? {
        return when (cell.cellType) {
            CellType.STRING ->
                cell.stringCellValue.trim().takeIf { it !in listOf("-", "", "`") }

            CellType.FORMULA -> {
                when (cell.cachedFormulaResultType) {
                    CellType.STRING -> cell.stringCellValue.trim().takeIf { it !in listOf("-", "", "`") }
                    CellType.NUMERIC -> cell.numericCellValue.toString().trim().takeIf { it !in listOf("-", "", "`") }
                    CellType.BOOLEAN -> cell.booleanCellValue.toString().trim().takeIf { it !in listOf("-", "", "`") }
                    else -> null
                }
            }

            CellType.NUMERIC ->
                cell.numericCellValue.toString().trim()
                    .takeIf { it !in listOf("-", "", "`") }

            else -> null
        }
    }

    private fun mapProcesses(
        processesMappingDtos: List<ProcessesMappingDto>,
        processCodesFromCell: List<String>,
        dictionaries: AgentImportDictionaries,
        rowNum: Int,
        fileId: Long
    ): MutableSet<ProcessEntity> {
        val resultProcesses = mutableSetOf<ProcessEntity>()

        processCodesFromCell.forEach { codeFromCell ->
            var dpssCode = findDpssCode(
                processesMappingDtos = processesMappingDtos,
                searchString = codeFromCell
            )

            if (dpssCode.isNullOrBlank()) {
                dpssCode = codeFromCell
            }

            val processRef = dictionaries.processesByShortName[dpssCode]
                ?: throw AiFileUploadException(
                    AI_UPLOAD_UNKNOWN_PROCESS_CODE,
                    messageProvider[AI_UPLOAD_UNKNOWN_PROCESS_CODE, codeFromCell, rowNum],
                    fileId = fileId
                )

            resultProcesses.add(
                managedReference(ProcessEntity::class.java, processRef.id)
            )
        }

        return resultProcesses
    }


    fun findDpssCode(processesMappingDtos: List<ProcessesMappingDto>, searchString: String): String? {
        return processesMappingDtos
            .firstOrNull { it.codeAris == searchString }
            ?.codeDpss
    }

    private fun getProcessCodesFromCell(cell: Cell): List<String> {
        val cellValue = getCellValueAsString(cell) ?: return emptyList()

        return cellValue.split(", ")
            .map { it.trim() }
            .filter { it.isNotBlank() && it !in listOf("-", "", "`") }
            .distinct()
    }

    private fun getCellValueAsString(cell: Cell): String? {
        return when (cell.cellType) {
            CellType.STRING ->
                cell.stringCellValue.trim().takeIf { it.isNotBlank() && it !in listOf("-", "`") }

            CellType.FORMULA -> {
                when (cell.cachedFormulaResultType) {
                    CellType.STRING -> cell.stringCellValue.trim().takeIf { it.isNotBlank() && it !in listOf("-", "`") }
                    CellType.NUMERIC -> cell.numericCellValue.toString().trim()
                        .takeIf { it.isNotBlank() && it !in listOf("-", "`") }

                    CellType.BOOLEAN -> cell.booleanCellValue.toString().trim()
                        .takeIf { it.isNotBlank() && it !in listOf("-", "`") }

                    else -> null
                }
            }

            CellType.NUMERIC ->
                cell.numericCellValue.toString().trim()
                    .takeIf { it.isNotBlank() && it !in listOf("-", "`") }

            else -> null
        }
    }

    private fun getValueOrNullFromFormula(cell: Cell): String? {
        return when {
            cell.cellType == CellType.FORMULA -> {
                when (cell.cachedFormulaResultType) {
                    CellType.STRING -> cell.stringCellValue.trim()
                    CellType.NUMERIC -> cell.numericCellValue.toString().trim()
                    CellType.BOOLEAN -> cell.booleanCellValue.toString().trim()
                    else -> null
                }
            }

            cell.cellType == CellType.STRING ->
                cell.stringCellValue.trim().takeIf { it !in listOf("-", "", "`") }

            cell.cellType == CellType.NUMERIC ->
                cell.numericCellValue.toString().trim()
                    .takeIf { it !in listOf("-", "", "`") }

            else -> null
        }
    }

    private fun checkDoublesAndAdd(
        aiAgent: String,
        processedAiAgents: MutableSet<String>,
        fileId: Long
    ) {
        if (processedAiAgents.contains(aiAgent)) {
            throw AiFileUploadException(
                errorCode = AI_UPLOAD_DOUBLES,
                message = MessageFormat.format(messageProvider[AI_UPLOAD_DOUBLES], aiAgent),
                fileId = fileId
            )
        }
        processedAiAgents.add(aiAgent)
    }

    private fun createException(exceptionType: String, row: Row, cell: Cell, fileId: Long): AiFileUploadException {
        return AiFileUploadException(
            errorCode = exceptionType,
            message = MessageFormat.format(messageProvider[exceptionType, cell.columnIndex.plus(1), row.rowNum.plus(1)]),
            fileId = fileId
        )
    }
}



@Component
class AgentImportDictionariesProvider(
    private val statusRepository: StatusRepository,
    private val metricRepository: MetricRepository,
    private val resourceRepository: ResourceRepository,
    private val divisionRepository: DivisionRepository,
    private val blockRepository: BlockRepository,
    private val processRepository: ProcessRepository,
    private val programRepository: ProgramRepository,
    private val initiativeTypeRepository: InitiativeTypeRepository,
    private val platformRepository: PlatformRepository,
    private val entityManager: EntityManager,
) {

    /**
     * Строит актуальный snapshot справочников для одного запуска импорта.
     *
     * Provider сам является singleton Spring-компонентом, но состояние между
     * загрузками не хранит. На каждый вызов создается новый immutable snapshot.
     */
    fun load(): AgentImportDictionaries {
        val statuses = statusRepository.findAll().map { status ->
            StatusRef(
                id = entityIdentifier(status) as Long,
                code = status.code,
                name = status.name,
                ordering = status.ordering
            )
        }

        val metrics = metricRepository.findAll().map { metric ->
            MetricRef(
                id = entityIdentifier(metric),
                fileName = metric.fileName
            )
        }

        val divisions = divisionRepository.findAll().map { division ->
            DivisionRef(
                id = entityIdentifier(division),
                code = division.code,
                blockId = division.block?.let(::entityIdentifier)
            )
        }

        val blocks = blockRepository.findAll().map { block ->
            BlockRef(
                id = entityIdentifier(block),
                shortName = block.shortName
            )
        }

        val processes = processRepository.findAll().map { process ->
            ProcessRef(
                id = entityIdentifier(process),
                shortName = process.shortName
            )
        }

        val programs = programRepository.findAll().map { program ->
            ProgramRef(
                id = entityIdentifier(program),
                fileName = program.fileName
            )
        }

        val initiativeTypes = initiativeTypeRepository.findAll().map { initiativeType ->
            InitiativeTypeRef(
                id = entityIdentifier(initiativeType),
                code = initiativeType.code
            )
        }

        val platforms = platformRepository.findAll().map { platform ->
            PlatformRef(
                id = entityIdentifier(platform) as Long,
                name = platform.name
            )
        }

        val resourcesByName = resourceRepository.findAll()
            .filter { !it.name.isNullOrBlank() }
            .map { resource ->
                ResourceRef(
                    name = resource.name!!,
                    source = resource.source,
                    type = resource.type
                )
            }
            .associateFirstBy { it.name }

        return AgentImportDictionaries(
            statusesByName = statuses
                .filter { !it.name.isNullOrBlank() }
                .associateFirstBy { normalizeKey(it.name!!) },

            statusesByCode = statuses
                .filter { !it.code.isNullOrBlank() }
                .associateFirstBy { normalizeKey(it.code!!) },

            statusesById = statuses.associateFirstBy { it.id },

            metricsByFileName = metrics
                .filter { !it.fileName.isNullOrBlank() }
                .associateFirstBy { normalizeKey(it.fileName!!) },

            resourcesByName = resourcesByName,

            divisionsByCode = divisions
                .filter { !it.code.isNullOrBlank() }
                .associateFirstBy { normalizeKey(it.code!!) },

            blocksByShortName = blocks
                .filter { !it.shortName.isNullOrBlank() }
                .associateFirstBy { normalizeKey(it.shortName!!) },

            processesByShortName = processes
                .filter { !it.shortName.isNullOrBlank() }
                .associateFirstBy { it.shortName!! },

            programsByFileName = programs
                .filter { !it.fileName.isNullOrBlank() }
                .associateFirstBy { normalizeKey(it.fileName!!) },

            initiativeTypesByCode = initiativeTypes
                .filter { !it.code.isNullOrBlank() }
                .associateFirstBy { it.code!! },

            platformsByName = platforms
                .filter { !it.name.isNullOrBlank() }
                .groupBy { it.name!! }
                .mapValues { (_, items) ->
                    items.maxByOrNull { it.id }!!
                }
        )
    }

    private fun entityIdentifier(entity: Any): Any =
        requireNotNull(
            entityManager.entityManagerFactory
                .persistenceUnitUtil
                .getIdentifier(entity)
        ) {
            "Entity identifier is null for ${entity.javaClass.simpleName}"
        }

    private fun normalizeKey(value: String): String =
        value.trim().lowercase()

    private inline fun <T, K> Iterable<T>.associateFirstBy(
        keySelector: (T) -> K
    ): Map<K, T> {
        val result = LinkedHashMap<K, T>()

        for (item in this) {
            result.putIfAbsent(keySelector(item), item)
        }

        return result
    }
}

data class AgentImportDictionaries(
    val statusesByName: Map<String, StatusRef>,
    val statusesByCode: Map<String, StatusRef>,
    val statusesById: Map<Long, StatusRef>,
    val metricsByFileName: Map<String, MetricRef>,
    val resourcesByName: Map<String, ResourceRef>,
    val divisionsByCode: Map<String, DivisionRef>,
    val blocksByShortName: Map<String, BlockRef>,
    val processesByShortName: Map<String, ProcessRef>,
    val programsByFileName: Map<String, ProgramRef>,
    val initiativeTypesByCode: Map<String, InitiativeTypeRef>,
    val platformsByName: Map<String, PlatformRef>
)

data class StatusRef(
    val id: Long,
    val code: String?,
    val name: String?,
    val ordering: Long?
)

data class MetricRef(
    val id: Any,
    val fileName: String?
)

data class ResourceRef(
    val name: String,
    val source: String?,
    val type: String?
)

data class DivisionRef(
    val id: Any,
    val code: String?,
    val blockId: Any?
)

data class BlockRef(
    val id: Any,
    val shortName: String?
)

data class ProcessRef(
    val id: Any,
    val shortName: String?
)

data class ProgramRef(
    val id: Any,
    val fileName: String?
)

data class InitiativeTypeRef(
    val id: Any,
    val code: String?
)

data class PlatformRef(
    val id: Long,
    val name: String?
)
```
