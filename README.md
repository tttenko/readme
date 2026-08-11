```java

@Query(
        """
        select
            a.id as id,
            a.agentId as agentId
        from AIAgentEntity a
        where a.agentId is not null
          and a.agentId not like '%PULT%'
        """
    )
    fun findAllNonPultAgentRefs(): List<AgentImportRefProjection>

    @Modifying
    @Query(
        """
        update AIAgentEntity a
        set a.disabled = true
        where a.id in :ids
        """
    )
    fun disableByIds(@Param("ids") ids: Collection<Long>): Int
}

interface AgentImportRefProjection {
    val id: Long
    val agentId: String
}

interface AgentContactRepository : JpaRepository<AgentContactEntity, Long> {

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
}

interface ContactRepository : JpaRepository<ContactEntity, Long> {

    fun findFirstByEmail(email: String): ContactEntity?

    fun findAllByEmailIn(emails: Collection<String>): List<ContactEntity>
}

@Modifying
    @Query(
        value = """
            with process_stats as (
                select
                    ap.process_id,
                    count(*) as total_count,
                    count(*) filter (where s.code = 'targetSolution') as target_count
                from agent_process ap
                join ai_agent a on a.id = ap.agent_id
                left join status s on s.id = a.agent_status_id
                group by ap.process_id
            ),
            highest_non_target as (
                select distinct on (ap.process_id)
                    ap.process_id,
                    s.id as status_id
                from agent_process ap
                join ai_agent a on a.id = ap.agent_id
                left join status s on s.id = a.agent_status_id
                where s.code is distinct from 'targetSolution'
                order by
                    ap.process_id,
                    s.ordering desc nulls last,
                    s.id desc nulls last
            ),
            resolved as (
                select
                    ps.process_id,
                    case
                        when ps.total_count = ps.target_count then (
                            select id
                            from status
                            where code = 'targetSolution'
                            order by id
                            limit 1
                        )
                        else h.status_id
                    end as status_id
                from process_stats ps
                left join highest_non_target h on h.process_id = ps.process_id
            )
            update process p
            set status_id = r.status_id
            from resolved r
            where p.id = r.process_id
        """,
        nativeQuery = true
    )
    fun refreshStatusesFromAgents(): Int
















@Suppress("UNUSED_EXPRESSION")
@Service
class AgentSheetService(
    private val aIAgentRepository: AIAgentRepository,
    private val statusRepository: StatusRepository,
    private val agentStatusSlaRepository: AgentStatusSlaRepository,
    private val divisionRepository: DivisionRepository,
    private val metricRepository: MetricRepository,
    private val blockRepository: BlockRepository,
    private val appliedMetricRepository: AppliedMetricRepository,
    private val resourceRepository: ResourceRepository,
    private val contactRepository: ContactRepository,
    private val platformRepository: PlatformRepository,
    private val programRepository: ProgramRepository,
    private val processRepository: ProcessRepository,
    private val fileUploadRepository: FileUploadRepository,
    private val initiativeTypeRepository: InitiativeTypeRepository,
    private val aiAgentQualityGateService: AiAgentQualityGateService,
    private val dateTimeProvider: DateTimeProvider,
    private val messageProvider: MessageProvider,
    private val transactionTemplate: TransactionTemplate,
    private val agentContactRepository: AgentContactRepository,
    private val entityManager: EntityManager,
) {
    private val log by logger()
    private val emailRegex = Regex("^[a-zA-Z0-9._%+-]+@(sberbank\\.ru|sber\\.ru)$")

    private companion object {
        const val AGENT_BATCH_SIZE = 100
        const val DB_OPERATION_BATCH_SIZE = 500
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

                // Удаление контактов теперь является частью общей транзакции импорта.
                // Если импорт упадет на любой строке, удаление тоже откатится.
                deleteNotPULTAgents()
                entityManager.flush()
                entityManager.clear()

                val agentsInFile = mutableSetOf<String>()
                val savedAgentIds = mutableListOf<Long>()
                val blockedAgents = mutableListOf<BlockedAgent>()
                val sheetStructure = getSheetStructure(agentSheet)

                val rowIterator = agentSheet.iterator()
                val batchRows = ArrayList<Row>(AGENT_BATCH_SIZE)
                var currentRowIndex = 0

                val duration = measureTimeMillis {
                    while (rowIterator.hasNext()) {
                        val row = rowIterator.next()
                        currentRowIndex++

                        if (currentRowIndex <= DATA_START_ROW_INDEX) {
                            continue
                        }

                        batchRows.add(row)

                        if (batchRows.size >= AGENT_BATCH_SIZE) {
                            savedAgentIds += processBatch(
                                rows = batchRows,
                                sheetStructure = sheetStructure,
                                context = context,
                                agentsInFile = agentsInFile,
                                blockedAgents = blockedAgents,
                                userId = userId
                            )
                            flushAndClear()
                            batchRows.clear()
                        }
                    }

                    if (batchRows.isNotEmpty()) {
                        savedAgentIds += processBatch(
                            rows = batchRows,
                            sheetStructure = sheetStructure,
                            context = context,
                            agentsInFile = agentsInFile,
                            blockedAgents = blockedAgents,
                            userId = userId
                        )
                        flushAndClear()
                        batchRows.clear()
                    }

                    if (agentsInFile.isEmpty()) {
                        throw AiFileUploadException(
                            errorCode = AI_UPLOAD_EMPTY_FILE,
                            message = MessageFormat.format(messageProvider[AI_UPLOAD_EMPTY_FILE]),
                            fileId = context.fileUploadId
                        )
                    }

                    refreshQualityGates(savedAgentIds)

                    disableDeletedAgents(agentsInFile)
                    updateProcessStatus()

                    val fileUploadEntity = fileUploadRepository.getReferenceById(context.fileUploadId)
                    fileUploadEntity.blockedAgents.clear()
                    fileUploadEntity.blockedAgents.addAll(blockedAgents)
                    fileUploadEntity.status = SUCCESS
                    fileUploadRepository.save(fileUploadEntity)
                }

                log.debug("Operation update entity took $duration ms")
                log.logSuccess(operationDetails)
            } catch (exception: Exception) {
                status.setRollbackOnly()
                throw exception
            }
        }
    }

    private fun processBatch(
        rows: List<Row>,
        sheetStructure: SheetStructure,
        context: DictAndProcessContext,
        agentsInFile: MutableSet<String>,
        blockedAgents: MutableList<BlockedAgent>,
        userId: Long
    ): List<Long> {
        val batchAgentIds = extractAgentIds(rows, sheetStructure.headers)
        val existingAgents: List<AIAgentEntity> = if (batchAgentIds.isEmpty()) {
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
            duplicatedAgentsToDisable.forEach {
                it.disabled = true
                it.updated = now
            }
            aIAgentRepository.saveAll(duplicatedAgentsToDisable)
        }

        val actualAgentIds = actualAgentsByAgentId.values.mapNotNull { it.id }
        val slasByAgentId: MutableMap<Long, MutableList<AgentStatusSlaEntity>> =
            if (actualAgentIds.isEmpty()) {
                mutableMapOf()
            } else {
            agentStatusSlaRepository.findAllByAiAgentIdIn(actualAgentIds)
                .filter { it.aiAgent?.id != null }
                .groupBy { it.aiAgent!!.id!! }
                .mapValues { (_, value) -> value.toMutableList() }
                .toMutableMap()
        }

        /*
         * EntityManager очищается после каждого batch, поэтому JPA-справочники
         * намеренно перечитываются для текущей пачки. Это не дает хранить
         * detached entity между batch и сохраняет корректную работу Hibernate.
         */
        val availableStatuses = statusRepository.findAll().toMutableList()
        val availableMetrics = metricRepository.findAll().toMutableList()
        val resourceEntities = resourceRepository.findAll()
        val availableDivisions = divisionRepository.findAll().toMutableList()
        val actualBlocks = blockRepository.findAll().toMutableList()
        val availableProcesses = processRepository.findAll().toMutableList()
        val availablePrograms = programRepository.findAll().toMutableList()
        val availableInitiativeTypes = initiativeTypeRepository.findAll().toMutableList()

        val contactEmails = extractContactEmails(rows, sheetStructure.headers)
        val availableContacts: MutableList<ContactEntity> = if (contactEmails.isEmpty()) {
            mutableListOf()
        } else {
            contactRepository.findAllByEmailIn(contactEmails).toMutableList()
        }

        val platformsByName: Map<String, PlatformEntity?> = platformRepository.findAll()
            .filter { !it.name.isNullOrBlank() }
            .groupBy { it.name!! }
            .mapValues { (_, items) -> items.maxByOrNull { it.id } }

        return rows.mapNotNull { row ->
            var savedId: Long? = null
            val oneEntityDuration = measureTimeMillis {
                savedId = processSingleRow(
                row = row,
                headers = sheetStructure.headers,
                headerRow = sheetStructure.headerRow,
                agentsInFile = agentsInFile,
                availableStatuses = availableStatuses,
                availableMetrics = availableMetrics,
                agentsByAgentId = actualAgentsByAgentId,
                slasByAgentId = slasByAgentId,
                availableInitiativeTypes = availableInitiativeTypes,
                availableContacts = availableContacts,
                resourceEntities = resourceEntities,
                availableDivisions = availableDivisions,
                actualBlocks = actualBlocks,
                availableProcesses = availableProcesses,
                availablePrograms = availablePrograms,
                context = context,
                fileUploadId = context.fileUploadId,
                blockedAgents = blockedAgents,
                platformsStartIndex = sheetStructure.platformsStartIndex,
                platformsEndIndex = sheetStructure.platformsEndIndex,
                metricsStartIndex = sheetStructure.metricsStartIndex,
                metricsEndIndex = sheetStructure.metricsEndIndex,
                resourcesStartIndex = sheetStructure.resourcesStartIndex,
                platformsByName = platformsByName,
                userId = userId
                )
            }

            log.debug("Operation update one entity took $oneEntityDuration ms")
            savedId
        }
    }

    private fun extractAgentIds(
        rows: List<Row>,
        headers: Map<String, Int>
    ): Set<String> {
        val agentIdColumnIndex = headers[AGENT_ID] ?: return emptySet()

        return rows.mapNotNull { row ->
            row.getCell(agentIdColumnIndex)?.let(::getValueOrNull)
        }.filter { it.isNotBlank() }
            .toSet()
    }

    private fun extractContactEmails(
        rows: List<Row>,
        headers: Map<String, Int>
    ): Set<String> {
        val contactColumns = listOfNotNull(
            headers[CUSTOMER_CONTACT],
            headers[DEVELOPER_CONTACT]
        )

        if (contactColumns.isEmpty()) return emptySet()

        return rows.asSequence()
            .flatMap { row ->
                contactColumns.asSequence().mapNotNull { columnIndex ->
                    row.getCell(columnIndex)
                        ?.let(::getValueOrNull)
                        ?.let(::extractContactEmail)
                }
            }
            .toSet()
    }

    private fun extractContactEmail(contact: String): String? {
        if (contact == "0") return null

        val email = contact.split(',', ';')
            .getOrNull(1)
            ?.takeIf { it.contains("@") }
            ?.removeSurrounding("<", ">")
            ?.trim()
            ?: return null

        return email.takeIf(emailRegex::matches)
    }

    private fun refreshQualityGates(savedAgentIds: List<Long>) {
        savedAgentIds
            .distinct()
            .chunked(DB_OPERATION_BATCH_SIZE)
            .forEach { agentIds ->
                aiAgentQualityGateService.createOrUpdateByAgent(agentIds)
                aiAgentQualityGateService.removeIrrelevant(agentIds)
            }
    }

    private fun flushAndClear() {
        entityManager.flush()
        entityManager.clear()
    }


    private fun selectActualAgent(agents: List<AIAgentEntity>): AIAgentEntity {
        return agents.sortedWith(
            compareBy<AIAgentEntity> { it.disabled == true }
                .thenByDescending { it.updated ?: it.created ?: LocalDateTime.MIN }
                .thenByDescending { it.id ?: 0L }
        ).first()
    }



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
        availableStatuses: MutableList<StatusEntity>,
        availableMetrics: MutableList<MetricEntity>,
        agentsByAgentId: MutableMap<String, AIAgentEntity>,
        slasByAgentId: MutableMap<Long, MutableList<AgentStatusSlaEntity>>,
        availableInitiativeTypes: MutableList<InitiativeTypeEntity>,
        availableContacts: MutableList<ContactEntity>,
        resourceEntities: List<ResourceEntity>,
        availableDivisions: MutableList<DivisionEntity>,
        actualBlocks: MutableList<BlockEntity>,
        availableProcesses: MutableList<ProcessEntity>,
        availablePrograms: MutableList<ProgramEntity>,
        context: DictAndProcessContext,
        fileUploadId: Long,
        blockedAgents: MutableList<BlockedAgent>,
        platformsStartIndex: Int,
        platformsEndIndex: Int,
        metricsStartIndex: Int,
        metricsEndIndex: Int,
        resourcesStartIndex: Int,
        platformsByName: Map<String, PlatformEntity?>,
        userId: Long
    ): Long? {
        val agentId = headers[AGENT_ID]?.takeIf { it >= 0 }?.let { index ->
            row.getCell(index)?.let { cell ->
                getValueOrNull(cell)
            }
        }

        if (agentId.isNullOrBlank()) return null

        checkDoublesAndAdd(agentId, agentsInFile, fileId = fileUploadId)

        var newEntity = false
        var entity = agentsByAgentId[agentId]?.also {
            if (it.importStatus == "blocked") {
                blockedAgents.add(BlockedAgent(it.id.toString(), it.agentName ?: ""))
                return null
            }
            it.agentStatus = updateAgentStatus(
                headers,
                row,
                availableStatuses,
                fileId = fileUploadId
            )
            it.updated = LocalDateTime.now()
        } ?: AIAgentEntity().also {
            it.agentId = agentId
            it.agentStatus = updateAgentStatus(
                headers,
                row,
                availableStatuses,
                fileId = fileUploadId
            )
            newEntity = true
        }

        updateAgentDivisionOrBlock(
            entity = entity,
            headers = headers,
            row = row,
            divisionsDict = context.divisionsDict,
            availableDivisions = availableDivisions,
            availableBlocks = actualBlocks,
            fileId = fileUploadId
        )

        val entityUpdateDuration = measureTimeMillis {
            updateBaseEntityFields(
                entity = entity,
                headers = headers,
                row = row,
                availableInitiativeTypes = availableInitiativeTypes,
                availableProcesses = availableProcesses,
                availablePrograms = availablePrograms,
                processesMappingDtos = context.processData.second,
                fileId = fileUploadId,
                availableContacts = availableContacts,
                userId = userId
            )
        }
        log.debug("Operation entity fields update took $entityUpdateDuration ms")

        entity = aIAgentRepository.save(entity)
        agentsByAgentId[agentId] = entity

        if (!newEntity) {
            appliedMetricRepository.deleteAllByAiAgentId(entity.id)
        }

        updateAppliedMetric(
            row = row,
            metricsStartIndex = metricsStartIndex,
            metricsEndIndex = metricsEndIndex,
            headerRow = headerRow,
            availableMetrics = availableMetrics,
            entity = entity,
            fileId = fileUploadId
        )

        val agentSlas = slasByAgentId.getOrPut(entity.id!!) { mutableListOf() }

        val pocMvpTargetDecisionMetricDuration = measureTimeMillis {
            updateImplementationStatusesOfAgent(
                entity,
                headers,
                row,
                availableStatuses,
                listOf(POC, MVP, TARGET_DECISION, CONFIRM_EFFECT),
                agentSlas
            )
        }
        log.debug("Operation pocMvpTargetDecision took $pocMvpTargetDecisionMetricDuration ms")

        updateInvolvedResource(
            row = row,
            headerRow = headerRow,
            resourcesStartIndex = resourcesStartIndex,
            resourcesEntities = resourceEntities,
            entity = entity,
            fileId = fileUploadId
        )

        updateAgentStatusSla(
            row = row,
            availableStatuses = availableStatuses,
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
                fileId = fileUploadId
            )
        }

        updateImplementedPlatform(
            platformsStartIndex = platformsStartIndex,
            platformsEndIndex = platformsEndIndex,
            headerRow = headerRow,
            row = row,
            entity = entity,
            platformsByName = platformsByName
        )

        return entity.id
    }

    private fun updateAgentDivisionOrBlock(
        entity: AIAgentEntity,
        headers: Map<String, Int>,
        row: Row,
        divisionsDict: List<DivisionsDto>,
        availableDivisions: MutableList<DivisionEntity>,
        availableBlocks: MutableList<BlockEntity>,
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

            val block = availableBlocks.find { it.shortName.equals(blockCellValue?.trim(), ignoreCase = true) }

            if (block == null) {
                throw AiFileUploadException(
                    AI_UPLOAD_EMPTY_TRIBE_NAME,
                    messageProvider[AI_UPLOAD_EMPTY_TRIBE_NAME, blockCellValue, row.rowNum.plus(1)],
                    fileId = fileId
                )
            }

            entity.division = null
            entity.block = block

            return
        }

        val division = divisionsDict
            .firstOrNull { it.shortName.equals(divisionCellValue.trim(), ignoreCase = true) }
            ?.let { divisionFromDict ->
                availableDivisions.find {
                    it.code.equals(divisionFromDict.code.trim(), ignoreCase = true)
                }
            }

        if (division == null) {
            throw AiFileUploadException(AI_UPLOAD_EMPTY_TRIBE_NAME,
                messageProvider[AI_UPLOAD_EMPTY_TRIBE_NAME, row.rowNum.plus(1)], fileId = fileId)
        }

        entity.division = division
        entity.block = division.block
    }

    private fun updateProcessStatus() {
        processRepository.refreshStatusesFromAgents()
    }

    private fun updateAgentStatus(
        headers: Map<String, Int>,
        row: Row,
        availableStatuses: MutableList<StatusEntity>,
        fileId: Long
    ): StatusEntity? {
        return headers[AGENT_STATUS]?.let { row.getCell(it) }?.let { statusCell ->
            val statusValue = getValueOrNullFromFormula(statusCell)
            availableStatuses.find {
                it.name.equals(
                    statusValue?.trim(),
                    ignoreCase = true
                )
            } ?: throw createException(
                exceptionType = AI_UPLOAD_UNKNOWN_AGENT_STATUS,
                row = row,
                cell = statusCell,
                fileId = fileId
            )
        }
    }

    private fun updateBaseEntityFields(
        entity: AIAgentEntity,
        headers: Map<String, Int>,
        row: Row,
        availableInitiativeTypes: MutableList<InitiativeTypeEntity>,
        availableProcesses: MutableList<ProcessEntity>,
        availablePrograms: MutableList<ProgramEntity>,
        processesMappingDtos: List<ProcessesMappingDto>,
        availableContacts: MutableList<ContactEntity>,
        fileId: Long,
        userId: Long
    ) {
        entity.also { currentEntity ->
            currentEntity.processes =
                headers[PROCESS_CODE]?.let {
                    row.getCell(it)?.let {
                        mapProcesses(
                            processesMappingDtos = processesMappingDtos,
                            processCodesFromCell = getProcessCodesFromCell(it),
                            availableProcesses = availableProcesses,
                            rowNum = it.rowIndex,
                            fileId = fileId,
                        )
                    }
                } ?: mutableSetOf()
            currentEntity.program = headers[PROGRAM]?.let {
                getValueOrNull(row.getCell(it))?.let { programId ->
                    availablePrograms.firstOrNull {
                        it.fileName.equals(programId, ignoreCase = true)
                    } ?: throw AiFileUploadException(
                        AI_UPLOAD_UNKNOWN_PROGRAM,
                        MessageFormat.format(
                            messageProvider[AI_UPLOAD_UNKNOWN_PROGRAM],
                            programId,
                            row.rowNum.plus(1)
                        ),
                        fileId = fileId
                    )
                }
            }
            currentEntity.agentName = headers[AGENT_NAME]?.let { columnIndex ->
                row.getCell(columnIndex)?.let { cell -> getValueOrNull(cell) } ?: throw AiFileUploadException(
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
                    row.getCell(columnIndex)?.let { cell -> getValueOrNull(cell)?.take(1000) }
                }
            currentEntity.agentProblem =
                headers[AGENT_PROBLEM]?.let { columnIndex ->
                    row.getCell(columnIndex)?.let { cell -> getValueOrNull(cell)?.take(1000) }
                }
            currentEntity.agentSolution =
                headers[AGENT_SOLUTION]?.let { columnIndex ->
                    row.getCell(columnIndex)?.let { cell -> getValueOrNull(cell)?.take(1000) }
                }
            currentEntity.agentJiraUrl =
                headers[AGENT_JIRA_URL]?.let { columnIndex ->
                    row.getCell(columnIndex)?.let { cell -> getValueOrNull(cell) }
                }
            currentEntity.initiativeType =
                headers[AGENT_INITIATIVE_TYPE]?.let { columnIndex ->
                    row.getCell(columnIndex)?.let { cell -> getValueOrNull(cell) }
                }?.let { fieldValue ->
                    if (fieldValue.lowercase().contains("агент")) {
                        availableInitiativeTypes.firstOrNull { it.code == "agent" }
                    } else {
                        availableInitiativeTypes.firstOrNull { it.code == "genAiSolution" }
                    }
                }

            val customerContact = headers[CUSTOMER_CONTACT]?.let { columnIndex ->
                row.getCell(columnIndex)?.let { cell ->
                    getValueOrNull(cell)?.let { contactValue ->
                        createContact(
                            contact = contactValue,
                            existingContacts = availableContacts,
                            agent = currentEntity,
                            userId = userId,
                            type = "customer"
                        )
                    }
                }
            }
            customerContact?.let { currentEntity.agentContact.add(it) }

            val developerContact = headers[DEVELOPER_CONTACT]?.let { columnIndex ->
                row.getCell(columnIndex)?.let { cell ->
                    getValueOrNull(cell)?.let { contactValue ->
                        createContact(
                            contact = contactValue,
                            existingContacts = availableContacts,
                            agent = currentEntity,
                            userId = userId,
                            type = "developer"
                        )
                    }
                }
            }
            developerContact?.let { currentEntity.agentContact.add(it) }

            currentEntity.agentEffectOptimization = headers[AGENT_EFFECT_OPTIMIZATION]?.let { columnIndex ->
                row.getCell(columnIndex)?.let { cell -> getValueOrNull(cell)?.toDouble()?.toBigDecimal() }
            }
            currentEntity.agentEffectRevenue = headers[AGENT_EFFECT_REVENUE]?.let { columnIndex ->
                row.getCell(columnIndex)?.let { cell -> getValueOrNull(cell)?.toDouble()?.toBigDecimal() }
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
        availableStatuses: MutableList<StatusEntity>,
        headerTypes: List<String>,
        agentSlas: MutableList<AgentStatusSlaEntity>
    ) {
        val slas = headerTypes.mapNotNull { headerType ->
            headers[headerType]
                ?.let { columnIndex -> row.getCell(columnIndex) }
                ?.let { cell ->
                    SlaData(headerType).apply {
                        if (cell.cellType == CellType.NUMERIC) dateValue = extractDateTime(cell)
                        if (cell.cellType == CellType.STRING) stringValue = cell.stringCellValue
                    }
                }
        }.associateBy { it.status }

        slas.forEach { sla ->
            val status = availableStatuses.find {
                it.code.equals(
                    when (sla.key) {
                        POC -> "analysis"
                        MVP -> "development"
                        TARGET_DECISION -> "pilot"
                        CONFIRM_EFFECT -> "targetSolution"
                        else -> null
                    }, ignoreCase = true
                )
            }

            val agentSla = agentSlas.find { it.agentStatus?.code == status?.code }

            if (sla.value.dateValue != null) {
                val saved = agentStatusSlaRepository.save(
                    (agentSla ?: AgentStatusSlaEntity().apply {
                        agentStatus = status
                        aiAgent = agent
                    }).apply {
                        plannedDate = sla.value.dateValue
                    }
                )
                agentSlas.removeIf { it.agentStatus?.code == saved.agentStatus?.code }
                agentSlas.add(saved)
            }

            if (sla.key == CONFIRM_EFFECT &&
                !slas[TARGET_DECISION]?.stringValue.equals("Завершен", ignoreCase = true)
            ) {
                agentSla?.apply { completedDate = null }?.let {
                    val saved = agentStatusSlaRepository.save(it)
                    agentSlas.removeIf { old -> old.agentStatus?.code == saved.agentStatus?.code }
                    agentSlas.add(saved)
                }
            }

            if (slas[TARGET_DECISION]?.stringValue.equals("Завершен", ignoreCase = true)
                && sla.key == CONFIRM_EFFECT
            ) {
                val saved = agentStatusSlaRepository.save(
                    (agentSla ?: AgentStatusSlaEntity().apply {
                        agentStatus = status
                        aiAgent = agent
                    }).apply {
                        completedDate = sla.value.dateValue ?: LocalDateTime.now()
                        plannedDate = this.plannedDate ?: sla.value.dateValue ?: LocalDateTime.now()
                    }
                )
                agentSlas.removeIf { it.agentStatus?.code == saved.agentStatus?.code }
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
        platformsByName: Map<String, PlatformEntity?>
    ) {
        val implementedPlatformDuration = measureTimeMillis {
            if (platformsStartIndex == -1 || platformsEndIndex == -1 || headerRow == null) return

            val entitiesToSave = row.filter { it.columnIndex in platformsStartIndex..platformsEndIndex }
                .mapNotNull { frontalCell ->
                    val cellHeader = headerRow.getCell(frontalCell.columnIndex)?.stringCellValue?.trim()
                        ?.takeIf { it.isNotBlank() }
                        ?: return@mapNotNull null

                    getValueOrNull(row.getCell(frontalCell.columnIndex))
                        ?.takeIf { it.isNotBlank() }
                        ?.let { rowFrontalValue ->
                            platformsByName[cellHeader]
                                ?.let { platformEntity ->
                                    if (setOf("в разработке", "внедрен").contains(rowFrontalValue.lowercase())) {
                                        ImplementedPlatformEntity().apply {
                                            primaryKey = AIAgentPlatformPK().apply {
                                                aiAgentId = entity.id
                                                platformId = platformEntity.id
                                            }
                                            platform = platformEntity
                                            aiAgent = entity
                                            released = rowFrontalValue.equals("Внедрен", ignoreCase = true)
                                        }
                                    } else null
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
        availableStatuses: MutableList<StatusEntity>,
        entity: AIAgentEntity,
        headers: Map<String, Int>,
        agentStatusesSla: List<AgentStatusSlaEntity>
    ) {
        val agentStatusSlaDuration = measureTimeMillis {
            headers[AGENT_STATUS]?.let { headerIndex ->
                var expirationDate: DeadlineExpiredType? = DeadlineExpiredType.ontime
                getValueOrNullFromFormula(row.getCell(headerIndex))?.let { status ->
                    val statusEntity =
                        availableStatuses.find { it.name.equals(status.trim(), ignoreCase = true) }
                    if (statusEntity != null) {
                        val filtredStatusSlas =
                            agentStatusesSla.filter { (it.agentStatus?.ordering ?: Long.MIN_VALUE) > (statusEntity.ordering ?: Long.MIN_VALUE) }
                        if (filtredStatusSlas.isNotEmpty()) {
                            filtredStatusSlas.forEach { statusSla ->
                                statusSla.plannedDate?.let { plannedDate ->
                                    if (plannedDate.toLocalDate().minusDays(14) < dateTimeProvider.currentDate()) {
                                        expirationDate = DeadlineExpiredType.expiration
                                    }
                                    if (plannedDate.toLocalDate() < dateTimeProvider.currentDate()) {
                                        expirationDate = DeadlineExpiredType.expired
                                    }
                                }
                            }
                        }
                    } else {
                        if (expirationDate != DeadlineExpiredType.expired && expirationDate != DeadlineExpiredType.expiration) {
                            expirationDate = DeadlineExpiredType.ontime
                        }
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
        resourcesEntities: List<ResourceEntity>,
        entity: AIAgentEntity,
        fileId: Long
    ) {
        val involvedResourceDuration = measureTimeMillis {
            if (resourcesStartIndex == -1 || headerRow == null) return

            val resourcesEndIndex = resourcesStartIndex + 3

            val entitiesToSave = row.filter { it.columnIndex in resourcesStartIndex..resourcesEndIndex }
                .mapNotNull { resource ->
                    val resourceName = headerRow.getCell(resource.columnIndex)?.stringCellValue?.trim()
                        ?.takeIf { it.isNotBlank() }
                        ?: return@mapNotNull null

                    getValueOrNull(row.getCell(resource.columnIndex))
                        ?.takeIf { it.isNotBlank() }
                        ?.let { rawValue ->
                            try {
                                val numericValue = rawValue.replace(',', '.').toBigDecimal()
                                resourcesEntities.firstOrNull { it.name == resourceName }?.let { resourceEntity ->
                                    InvolvedResourceEntity().apply {
                                        this.id = InvolvedResourceEmbeddedId(
                                            aiAgentId = entity.id,
                                            source = resourceEntity.source,
                                            type = resourceEntity.type,
                                        )
                                        this.value = numericValue
                                        this.aiAgent = entity
                                        this.updated = LocalDateTime.now()
                                    }
                                } ?: throw AiFileUploadException(
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
        availableMetrics: MutableList<MetricEntity>,
        entity: AIAgentEntity,
        fileId: Long
    ) {
        val appliedMetricDuration = measureTimeMillis {
            if (metricsStartIndex == -1 || metricsEndIndex == -1 || headerRow == null) return

            val metricsToSave = mutableListOf<AppliedMetricsEntity>()

            row.filter { it.columnIndex in metricsStartIndex..metricsEndIndex }
                .forEach { metricValue ->
                    val metricName = headerRow.getCell(metricValue.columnIndex)?.stringCellValue?.trim()
                    if (metricName.equals("Индивидуальная метрика", ignoreCase = true)) {
                        return@forEach
                    }
                    getValueOrNull(row.getCell(metricValue.columnIndex))
                        ?.takeIf { it.isNotBlank() }
                        ?.let { rawValue ->
                            try {
                                val normalizedValue = rawValue.replace(',', '.').toDouble()
                                availableMetrics.firstOrNull {
                                    it.fileName.equals(metricName, ignoreCase = true)
                                }?.let { existingMetric ->
                                    metricsToSave.add(
                                        AppliedMetricsEntity().also {
                                            it.aiAgent = entity
                                            it.metric = existingMetric
                                            it.currentValue = BigDecimal.valueOf(normalizedValue)
                                            it.type = STANDARD
                                            it.updated = LocalDateTime.now()
                                        }
                                    )
                                } ?: throw AiFileUploadException(
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
        val agentIdsToDisable = aIAgentRepository.findAllNonPultAgentRefs()
            .asSequence()
            .filter { it.agentId !in agentsInFile }
            .map { it.id }
            .toList()

        agentIdsToDisable
            .chunked(DB_OPERATION_BATCH_SIZE)
            .forEach(aIAgentRepository::disableByIds)
    }

    private fun deleteNotPULTAgents() {
        agentContactRepository.deleteAllForImportedNonBlockedAgents()
    }

    private fun createContact(
        contact: String,
        existingContacts: MutableList<ContactEntity>,
        agent: AIAgentEntity?,
        type: String,
        userId: Long
    ): AgentContactEntity? {
        if (contact == "0") return null
        else contact.split(',', ';').let { fields ->
            val email = fields.getOrNull(1)
            val contactEntity = email?.let {
                if (it.contains("@")) {
                    val pureEmail = email.removeSurrounding("<", ">").trim()
                    if (!emailRegex.matches(pureEmail)) {
                        return null
                    }
                    val existingContact = existingContacts.firstOrNull { contact -> contact.email == pureEmail }
                    existingContact?.apply {
                        this.fio = fields.getOrNull(0)
                    } ?: createAndSaveContactEntity(existingContacts, fields, pureEmail)
                } else {
                    return null
                }
            } ?: return null
            return AgentContactEntity(
                agent = agent,
                type = type,
                contact = contactEntity
            )
        }
    }

    private fun createAndSaveContactEntity(
        existingContacts: MutableList<ContactEntity>,
        fields: List<String>,
        pureEmail: String,
    ): ContactEntity {
        val contactEntity = contactRepository.save(ContactEntity().apply {
            this.fio = fields.getOrNull(0)
            this.email = pureEmail
        })
        existingContacts.add(contactEntity)
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
        availableProcesses: MutableList<ProcessEntity>,
        rowNum: Int,
        fileId: Long
    ): MutableSet<ProcessEntity> {
        val resultProcessesIds = mutableSetOf<ProcessEntity>()
        processCodesFromCell.forEach { codeFormCell ->
            var dpssCode = findDpssCode(processesMappingDtos = processesMappingDtos, searchString = codeFormCell)
            if (dpssCode.isNullOrBlank()) {
                dpssCode = codeFormCell
            }
            resultProcessesIds.add(availableProcesses.firstOrNull { it.shortName == dpssCode }
                ?: throw AiFileUploadException(
                    AI_UPLOAD_UNKNOWN_PROCESS_CODE,
                    messageProvider[AI_UPLOAD_UNKNOWN_PROCESS_CODE, codeFormCell, rowNum],
                    fileId = fileId
                ))
        }
        return resultProcessesIds
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
```
