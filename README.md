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

    @Transactional(propagation = Propagation.MANDATORY)
    fun processAgentSheet(
        context: DictAndProcessContext,
        agentSheet: PreparedSheet,
        userId: Long
    ) {
        log.info("Agent import started. fileUploadId=${context.fileUploadId}")

        val operationDetails = "Request 'process Agent page' with user=$userId "

        transactionTemplate.execute { status ->
            try {
                log.logBefore(operationDetails)

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
                    agentSheet.readRows { rowIterator ->
                        val batch = ArrayList<PreparedRow>(AGENT_BATCH_SIZE)
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
                    } // освобождаем дескриптор подготовленного листа

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

                // Сохраняем старое глобальное правило удаления контактов отсутствующих,
                // дублей и других не-PULT/не-blocked агентов, но не удаляем
                // контакты обработанных агентов ради последующего пересоздания.
                clearContactsForUnprocessedAgents(savedAgentIds.toSet())
                disableDeletedAgents(agentsInFile)

                /*
                 * Не загружаем process.agents целиком.
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
                fileUploadEntity.errorDescription = null
                fileUploadEntity.updatedBy = userId
                fileUploadEntity.uploaded = LocalDateTime.now()
                // @PreUpdate установит updated при flush в общей транзакции.
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
        rows: List<PreparedRow>,
        sheetStructure: SheetStructure,
        dictionaries: AgentImportDictionaries,
        context: DictAndProcessContext,
        agentsInFile: MutableSet<String>,
        blockedAgents: MutableList<BlockedAgent>,
        userId: Long
    ): List<Long> {

        log.info(
            "Processing agent import batch: size={}, firstRow={}, lastRow={}",
            rows.size, rows.firstOrNull()?.rowNum, rows.lastOrNull()?.rowNum
        )

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

        // Два fetch-запроса гарантируют, что managed inverse-коллекции Hibernate
        // инициализированы ДО обхода каждой строки. Никакого N+1 в iterator().
        if (actualAgentIds.isNotEmpty()) {
            aIAgentRepository.fetchPlatformsForExcelImport(actualAgentIds)
            aIAgentRepository.fetchResourcesForExcelImport(actualAgentIds)
            aIAgentRepository.fetchContactsForExcelImport(actualAgentIds)
        }
        val existingPlatformsByAgentId = actualAgentsByAgentId.values
            .associate { current ->
                current.id to current.platforms
                    .mapNotNull { platform ->
                        platform.primaryKey.platformId?.let { it to platform }
                    }.toMap()
            }
        val existingResourcesByAgentId = actualAgentsByAgentId.values
            .associate { current -> current.id to current.involvedResource.toList() }

        val existingMetricsByAgentId: Map<Long, List<AppliedMetricsEntity>> =
            if (actualAgentIds.isEmpty()) emptyMap()
            else appliedMetricRepository.findAllByAiAgentIdIn(actualAgentIds)
                .groupBy { it.aiAgent!!.id }

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
                    existingPlatformsByAgentId = existingPlatformsByAgentId,
                    existingResourcesByAgentId = existingResourcesByAgentId,
                    existingMetricsByAgentId = existingMetricsByAgentId,
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

        log.info("Agent import batch completed: size={}, saved={}", rows.size, savedIds.size)

        return savedIds
    }

    private fun loadContactsForBatch(
        rows: List<PreparedRow>,
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


    private fun getSheetStructure(sheet: PreparedSheet): SheetStructure {
        return sheet.readRows { iterator ->
            var currentRowIndex = 0
            var headers = emptyMap<String, Int>()
            var platformsStartIndex = -1
            var platformsEndIndex = -1
            var metricsStartIndex = -1
            var metricsEndIndex = -1
            var resourcesStartIndex = -1
            var headerRow: PreparedRow? = null

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

            SheetStructure(
                headers = headers,
                headerRow = headerRow,
                platformsStartIndex = platformsStartIndex,
                platformsEndIndex = platformsEndIndex,
                metricsStartIndex = metricsStartIndex,
                metricsEndIndex = metricsEndIndex,
                resourcesStartIndex = resourcesStartIndex
            )
        }
    }

    private data class SheetStructure(
        val headers: Map<String, Int>,
        val headerRow: PreparedRow?,
        val platformsStartIndex: Int,
        val platformsEndIndex: Int,
        val metricsStartIndex: Int,
        val metricsEndIndex: Int,
        val resourcesStartIndex: Int
    )

    private fun processSingleRow(
        row: PreparedRow,
        headers: Map<String, Int>,
        headerRow: PreparedRow?,
        agentsInFile: MutableSet<String>,
        dictionaries: AgentImportDictionaries,
        agentsByAgentId: MutableMap<String, AIAgentEntity>,
        slasByAgentId: MutableMap<Long, MutableList<AgentStatusSlaEntity>>,
        contactsByEmail: MutableMap<String, ContactEntity>,
        existingPlatformsByAgentId: Map<Long, Map<Long, ImplementedPlatformEntity>>,
        existingResourcesByAgentId: Map<Long, List<InvolvedResourceEntity>>,
        existingMetricsByAgentId: Map<Long, List<AppliedMetricsEntity>>,
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
                    BlockedAgent(agentId = existing.id.toString(), agentName = existing.agentName ?: "")
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

        if (newEntity) {
            entity = aIAgentRepository.save(entity)
        }

        agentsByAgentId[agentId] = entity

        // Этап 2: точечный diff вместо DELETE-all + INSERT (см. updateAppliedMetric).
        updateAppliedMetric(
            row = row,
            metricsStartIndex = metricsStartIndex,
            metricsEndIndex = metricsEndIndex,
            headerRow = headerRow,
            dictionaries = dictionaries,
            entity = entity,
            fileId = fileId,
            existingMetrics = existingMetricsByAgentId[entity.id].orEmpty()
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
            fileId = fileId,
            existingResources = existingResourcesByAgentId[entity.id]
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
            dictionaries = dictionaries,
            existingByPlatformId = existingPlatformsByAgentId[entity.id]
        )

        return entity.id
    }

    private fun updateAgentDivisionOrBlock(
        entity: AIAgentEntity,
        headers: Map<String, Int>,
        row: PreparedRow,
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
        row: PreparedRow,
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
        row: PreparedRow,
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
                    row.getCell(columnIndex)?.let(::getValueOrNull)?.take(2000)
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
            synchronizeAgentContacts(
                agent = currentEntity,
                desiredContacts = listOfNotNull(customerContact, developerContact)
            )

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
        row: PreparedRow,
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

            val statusRef = statusCode?.let(::normalizeKey)?.let(dictionaries.statusesByCode::get)

            val agentSla = statusRef?.let { ref -> agentSlas.find { it.primaryKey.agentStatusId == ref.id } }

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

            if (sla.key == CONFIRM_EFFECT && !slas[TARGET_DECISION]?.stringValue.equals(
                    "Завершен",
                    ignoreCase = true
                )
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

                agentSlas.removeIf { it.primaryKey.agentStatusId == saved.primaryKey.agentStatusId }
                agentSlas.add(saved)
            }
        }
    }


    private fun extractDateTime(cell: PreparedCell?): LocalDateTime? {
        if (cell == null) return null

        return when (cell.cellType) {
            CellType.NUMERIC -> {
                if (cell.isDateFormatted) {
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
                if (cell.cachedFormulaResultType == CellType.NUMERIC && cell.isDateFormatted) {
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
        headerRow: PreparedRow?,
        row: PreparedRow,
        entity: AIAgentEntity,
        dictionaries: AgentImportDictionaries,
        existingByPlatformId: Map<Long, ImplementedPlatformEntity>?
    ) {
        val implementedPlatformDuration = measureTimeMillis {
            if (platformsStartIndex == -1 || platformsEndIndex == -1 || headerRow == null) {
                return@measureTimeMillis
            }

            val agentId = requireNotNull(entity.id) {
                "Agent id must be initialized before implemented platforms update"
            }

            val desiredPlatforms = linkedMapOf<Long, Boolean>()

            row.filter { it.columnIndex in platformsStartIndex..platformsEndIndex }
                .forEach { platformCell ->
                    val cellHeader = headerRow
                        .getCell(platformCell.columnIndex)
                        ?.stringCellValue
                        ?.trim()
                        ?.takeIf { it.isNotBlank() }
                        ?: return@forEach

                    val rawValue = getValueOrNull(row.getCell(platformCell.columnIndex))
                        ?.takeIf { it.isNotBlank() }
                        ?: return@forEach

                    val platformRef = dictionaries.platformsByName[cellHeader] ?: return@forEach

                    val released = when {
                        rawValue.equals("Внедрен", ignoreCase = true) -> true
                        rawValue.equals("В разработке", ignoreCase = true) -> false
                        else -> return@forEach
                    }

                    desiredPlatforms[platformRef.id] = released
                }

            // Этап 3: база для сверки приходит пакетно (из processBatch), без отдельной
            // загрузки inverse-коллекции. Добавления/удаления по-прежнему идут через
            // entity.platforms ради orphan-removal/cascade.
            val iterator = entity.platforms.iterator()

            while (iterator.hasNext()) {
                val existing = iterator.next()

                if (existing.primaryKey.platformId !in desiredPlatforms) {
                    iterator.remove()
                }
            }

            desiredPlatforms.forEach { (platformId, released) ->

                val existing =
                    existingByPlatformId?.get(platformId)

                if (existing != null) {
                    existing.released = released
                    existing.aiAgent = entity
                } else {
                    entity.platforms.add(
                        ImplementedPlatformEntity().apply {
                            primaryKey =
                                AIAgentPlatformPK().apply {
                                    aiAgentId = agentId
                                    this.platformId = platformId
                                }

                            platform = managedReference(PlatformEntity::class.java, platformId)
                            aiAgent = entity
                            this.released = released
                        }
                    )
                }
            }
        }

        log.debug(
            "Operation implemented platforms took $implementedPlatformDuration ms"
        )
    }


    private fun updateTerBanks(
        row: PreparedRow,
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
        row: PreparedRow,
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
        row: PreparedRow,
        headerRow: PreparedRow?,
        resourcesStartIndex: Int,
        dictionaries: AgentImportDictionaries,
        entity: AIAgentEntity,
        fileId: Long,
        existingResources: List<InvolvedResourceEntity>?
    ) {
        val involvedResourceDuration = measureTimeMillis {
            if (resourcesStartIndex == -1 || headerRow == null) {
                return@measureTimeMillis
            }

            val agentId = requireNotNull(entity.id) {
                "Agent id must be initialized before involved resources update"
            }

            val resourcesEndIndex = resourcesStartIndex + 3

            val desiredResources = linkedMapOf<Pair<String?, String?>, BigDecimal>()

            row.filter { it.columnIndex in resourcesStartIndex..resourcesEndIndex }
                .forEach { resourceCell ->
                    val resourceName = headerRow
                        .getCell(resourceCell.columnIndex)
                        ?.stringCellValue
                        ?.trim()
                        ?.takeIf { it.isNotBlank() }
                        ?: return@forEach

                    val rawValue = getValueOrNull(row.getCell(resourceCell.columnIndex))
                        ?.takeIf { it.isNotBlank() }
                        ?: return@forEach

                    val resourceRef = dictionaries.resourcesByName[resourceName]
                        ?: throw AiFileUploadException(
                            AI_UPLOAD_UNKNOWN_METRIC_NAME,
                            MessageFormat.format(
                                messageProvider[AI_UPLOAD_UNKNOWN_METRIC_NAME],
                                resourceCell.columnIndex.plus(1),
                                row.rowNum.plus(1)
                            ),
                            fileId = fileId
                        )

                    val numericValue = try {
                        rawValue
                            .replace(',', '.')
                            .toBigDecimal()
                    } catch (e: NumberFormatException) {
                        throw AiFileUploadException(
                            AI_UPLOAD_UNKNOWN_METRIC_NAME,
                            MessageFormat.format(
                                messageProvider[AI_UPLOAD_UNKNOWN_METRIC_NAME],
                                resourceCell.columnIndex.plus(1),
                                row.rowNum.plus(1)
                            ),
                            fileId = fileId
                        )
                    }

                    val key =
                        resourceRef.source to resourceRef.type

                    desiredResources[key] = numericValue
                }

            // Этап 3: база для сверки приходит пакетно (из processBatch), без отдельной
            // загрузки inverse-коллекции involvedResource.
            val existingByKey = existingResources
                ?.associateBy { resource -> resource.id.source to resource.id.type }
                .orEmpty()

            val iterator = entity.involvedResource.iterator()

            while (iterator.hasNext()) {
                val existing = iterator.next()

                val key = existing.id.source to existing.id.type

                if (key !in desiredResources) {
                    iterator.remove()
                }
            }

            val now = LocalDateTime.now()

            desiredResources.forEach { (key, value) ->

                val existing = existingByKey[key]

                if (existing != null) {
                    existing.value = value
                    existing.updated = now
                    existing.aiAgent = entity
                } else {
                    entity.involvedResource.add(
                        InvolvedResourceEntity().apply {
                            id = InvolvedResourceEmbeddedId(aiAgentId = agentId, source = key.first, type = key.second)
                            this.value = value
                            aiAgent = entity
                            updated = now
                        }
                    )
                }
            }
        }

        log.debug(
            "Operation involved Resource took $involvedResourceDuration ms"
        )
    }

    /**
     * Сверка всего набора метрик, а не SELECT/DELETE по одной метрике на агента.
     * Сохраняет старое правило: строки всех типов, отсутствующие в целевом STANDARD-наборе,
     * удаляются. Поддерживает несколько строк одного metric.id (без associateBy-потерь).
     */
    private fun updateAppliedMetric(
        row: PreparedRow,
        metricsStartIndex: Int,
        metricsEndIndex: Int,
        headerRow: PreparedRow?,
        dictionaries: AgentImportDictionaries,
        entity: AIAgentEntity,
        fileId: Long,
        existingMetrics: List<AppliedMetricsEntity>
    ) {
        if (metricsStartIndex == -1 || metricsEndIndex == -1 || headerRow == null) {
            // Как в старом обработчике: отсутствие секции метрик означает пустой
            // импортируемый набор; прежние метрики этого агента удаляются.
            if (existingMetrics.isNotEmpty()) {
                appliedMetricRepository.deleteAll(existingMetrics)
            }
            return
        }

        val reusableByMetricId = existingMetrics
            .filter { it.type == STANDARD && it.metric?.id != null }
            .groupBy { it.metric!!.id }
            .mapValues { (_, matches) -> ArrayDeque(matches) }
        val retainedIds = mutableSetOf<Long>()
        val newMetrics = mutableListOf<AppliedMetricsEntity>()

        row.filter { it.columnIndex in metricsStartIndex..metricsEndIndex }.forEach { metricCell ->
            val metricName = headerRow.getCell(metricCell.columnIndex)?.stringCellValue?.trim()
            if (metricName.equals("Индивидуальная метрика", ignoreCase = true)) return@forEach
            val rawValue = getValueOrNull(metricCell)?.takeIf { it.isNotBlank() } ?: return@forEach
            val metricRef = metricName?.let(::normalizeKey)?.let(dictionaries.metricsByFileName::get)
                ?: throw AiFileUploadException(
                    AI_UPLOAD_UNKNOWN_METRIC_NAME,
                    MessageFormat.format(
                        messageProvider[AI_UPLOAD_UNKNOWN_METRIC_NAME],
                        metricCell.columnIndex + 1, row.rowNum + 1
                    ), fileId = fileId
                )
            val amount = try {
                BigDecimal.valueOf(rawValue.replace(',', '.').toDouble())
            } catch (exception: NumberFormatException) {
                throw AiFileUploadException(
                    AI_UPLOAD_UNKNOWN_METRIC_NAME,
                    MessageFormat.format(
                        messageProvider[AI_UPLOAD_UNKNOWN_METRIC_NAME],
                        metricCell.columnIndex + 1, row.rowNum + 1
                    ), fileId = fileId
                )
            }
            val metricId = (metricRef.id as Number).toLong()
            val current = reusableByMetricId[metricId]?.pollFirst()
            if (current == null) {
                newMetrics.add(AppliedMetricsEntity().apply {
                    aiAgent = entity
                    metric = managedReference(MetricEntity::class.java, metricId)
                    currentValue = amount
                    type = STANDARD
                    updated = LocalDateTime.now()
                })
            } else {
                retainedIds.add(current.id)
                // Прежняя операция DELETE+INSERT обнуляла эти поля;
                // сохраняем итоговое бизнес-состояние, но не меняем PK без причины.
                val changed = current.currentValue != amount || current.name != null ||
                        current.dataType != null || current.direction != null || current.targetValue != null
                if (changed) {
                    current.currentValue = amount
                    current.name = null
                    current.dataType = null
                    current.direction = null
                    current.targetValue = null
                    current.updated = LocalDateTime.now()
                }
            }
        }
        val obsolete = existingMetrics.filter { it.id !in retainedIds }
        if (obsolete.isNotEmpty()) appliedMetricRepository.deleteAll(obsolete)
        if (newMetrics.isNotEmpty()) appliedMetricRepository.saveAll(newMetrics)
    }

    private fun disableDeletedAgents(agentsInFile: Set<String>) {
        aIAgentRepository.findAllImportManagedNonPultAgentRefs()
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

    /**
     * Прежний глобальный DELETE затрагивал всех не-PULT и не-blocked агентов,
     * в том числе отсутствующих в файле и дублирующие записи.
     * Теперь нетронутые контакты выбранных агентов сохраняют ID; остальные
     * глобально очищаются отдельными bulk DELETE только ПОСЛЕ обработки файла.
     */
    private fun clearContactsForUnprocessedAgents(processedAgentIds: Set<Long>) {
        aIAgentRepository.findAllImportManagedNonPultAgentRefs()
            .asSequence()
            .map { it.id }
            .filterNot(processedAgentIds::contains)
            .chunked(BULK_BATCH_SIZE)
            .forEach { ids ->
                if (ids.isNotEmpty()) agentContactRepository.deleteForAgentIds(ids)
            }
    }

    /**
     * Сверяет ссылки контактов без удаления/вставки неизменённых строк.
     *
     * Для non-PULT удаляем только устаревшие и повторяющиеся ссылки.
     * userId у импортируемой ссылки приводим к null, как после старой
     * глобальной очистки и повторного создания: это влияет на фильтр
     * AIAgentRepository по agent_contact.user_id.
     *
     * Для PULT сохраняем чужие, отсутствующие в Excel ссылки: старая
     * глобальная очистка их не затрагивала. Добавляем только отсутствующие,
     * исключая размножение одинаковых строк при повторном импорте.
     */
    private fun synchronizeAgentContacts(
        agent: AIAgentEntity,
        desiredContacts: List<AgentContactEntity>
    ) {
        val desiredByKey = linkedMapOf<Pair<String?, Long?>, AgentContactEntity>()
        desiredContacts.forEach { desired ->
            desiredByKey.putIfAbsent(desired.type to desired.contact?.id, desired)
        }

        val isPultAgent = agent.agentId?.contains("PULT") == true
        val retainedKeys = mutableSetOf<Pair<String?, Long?>>()
        val existingIterator = agent.agentContact.iterator()

        while (existingIterator.hasNext()) {
            val existing = existingIterator.next()
            val key = existing.type to existing.contact?.id
            val importedContact = key in desiredByKey

            if (isPultAgent) {
                // Чужие/старые PULT-связи не удаляем. Даже если существовали
                // дубли, не создаём поверх них новую копию.
                if (importedContact) retainedKeys.add(key)
                continue
            }

            if (!importedContact || !retainedKeys.add(key)) {
                existingIterator.remove()
                continue
            }

            // Прежний non-PULT импорт создавал ссылку с userId=null.
            // Сохраняем то же значение без нового id / физического пересоздания.
            if (existing.userId != null) {
                existing.userId = null
            }
        }

        desiredByKey.forEach { (key, newLink) ->
            if (key !in retainedKeys) {
                agent.agentContact.add(newLink)
            }
        }
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

        return AgentContactEntity(agent = agent, type = type, contact = contactEntity)
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


    private fun getValueOrNull(cell: PreparedCell?): String? {
        if (cell == null) return null
        return when (cell.cellType) {
            CellType.STRING -> cell.stringCellValue.trim().takeIf { it !in listOf("-", "", "`") }

            CellType.FORMULA -> {
                when (cell.cachedFormulaResultType) {
                    CellType.STRING -> cell.stringCellValue.trim().takeIf { it !in listOf("-", "", "`") }
                    CellType.NUMERIC -> cell.numericCellValue.toString().trim().takeIf { it !in listOf("-", "", "`") }
                    CellType.BOOLEAN -> cell.booleanCellValue.toString().trim().takeIf { it !in listOf("-", "", "`") }
                    else -> null
                }
            }

            CellType.NUMERIC -> cell.numericCellValue.toString().trim().takeIf { it !in listOf("-", "", "`") }

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
            var dpssCode = findDpssCode(processesMappingDtos = processesMappingDtos, searchString = codeFromCell)

            if (dpssCode.isNullOrBlank()) {
                dpssCode = codeFromCell
            }

            val processRef = dictionaries.processesByShortName[dpssCode]
                ?: throw AiFileUploadException(
                    AI_UPLOAD_UNKNOWN_PROCESS_CODE,
                    messageProvider[AI_UPLOAD_UNKNOWN_PROCESS_CODE, codeFromCell, rowNum],
                    fileId = fileId
                )

            resultProcesses.add(managedReference(ProcessEntity::class.java, processRef.id))
        }

        return resultProcesses
    }


    fun findDpssCode(processesMappingDtos: List<ProcessesMappingDto>, searchString: String): String? {
        return processesMappingDtos.firstOrNull { it.codeAris == searchString }?.codeDpss
    }

    private fun getProcessCodesFromCell(cell: PreparedCell): List<String> {
        val cellValue = getCellValueAsString(cell) ?: return emptyList()

        return cellValue.split(", ")
            .map { it.trim() }
            .filter { it.isNotBlank() && it !in listOf("-", "", "`") }
            .distinct()
    }

    private fun getCellValueAsString(cell: PreparedCell?): String? {
        if (cell == null) return null
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

    private fun getValueOrNullFromFormula(cell: PreparedCell?): String? {
        if (cell == null) return null
        return when {
            cell.cellType == CellType.FORMULA -> {
                when (cell.cachedFormulaResultType) {
                    CellType.STRING -> cell.stringCellValue.trim()
                    CellType.NUMERIC -> cell.numericCellValue.toString().trim()
                    CellType.BOOLEAN -> cell.booleanCellValue.toString().trim()
                    else -> null
                }
            }

            cell.cellType == CellType.STRING -> cell.stringCellValue.trim().takeIf { it !in listOf("-", "", "`") }
            cell.cellType == CellType.NUMERIC -> cell.numericCellValue.toString().trim()
                .takeIf { it !in listOf("-", "", "`") }

            else -> null
        }
    }

    private fun checkDoublesAndAdd(aiAgent: String, processedAiAgents: MutableSet<String>, fileId: Long) {
        if (processedAiAgents.contains(aiAgent)) {
            throw AiFileUploadException(
                errorCode = AI_UPLOAD_DOUBLES,
                message = MessageFormat.format(messageProvider[AI_UPLOAD_DOUBLES], aiAgent),
                fileId = fileId
            )
        }
        processedAiAgents.add(aiAgent)
    }

    private fun createException(exceptionType: String, row: PreparedRow, cell: PreparedCell, fileId: Long): AiFileUploadException {
        return AiFileUploadException(
            errorCode = exceptionType,
            message = MessageFormat.format(messageProvider[exceptionType, cell.columnIndex.plus(1), row.rowNum.plus(1)]),
            fileId = fileId
        )
    }

}

```
