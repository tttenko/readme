```java

@ExtendWith(MockKExtension::class)
class AgentSheetServiceTest {

    @MockK
    private lateinit var aIAgentRepository: AIAgentRepository

    @MockK
    private lateinit var agentStatusSlaRepository: AgentStatusSlaRepository

    @MockK
    private lateinit var appliedMetricRepository: AppliedMetricRepository

    @MockK
    private lateinit var contactRepository: ContactRepository

    @MockK
    private lateinit var processRepository: ProcessRepository

    @MockK
    private lateinit var fileUploadRepository: FileUploadRepository

    @MockK
    private lateinit var aiAgentQualityGateService: AiAgentQualityGateService

    @MockK
    private lateinit var dateTimeProvider: DateTimeProvider

    @MockK(relaxed = true)
    private lateinit var messageProvider: MessageProvider

    @MockK
    private lateinit var transactionTemplate: TransactionTemplate

    @MockK
    private lateinit var agentContactRepository: AgentContactRepository

    @MockK(relaxed = true)
    private lateinit var entityManager: EntityManager

    @MockK
    private lateinit var dictionariesProvider: AgentImportDictionariesProvider

    @InjectMockKs
    private lateinit var agentSheetService: AgentSheetService

    private lateinit var mockSheet: Sheet
    private lateinit var context: DictAndProcessContext
    private lateinit var fileUploadEntity: FileUploadEntity
    private lateinit var transactionStatus: TransactionStatus

    private var nextAgentId = 1L

    @BeforeEach
    fun setUp() {
        mockSheet = mockk()

        fileUploadEntity = FileUploadEntity().apply {
            id = 1L
            status = FileUploadStatus.PROCESSING
        }

        context = DictAndProcessContext(
            fileUploadId = 1L,
            divisionsDict = listOf(
                DivisionsDto("DIV001", "Division 1", "DIV001", "block", "name"),
                DivisionsDto("DIV002", "Division 2", "DIV002", "block", "name")
            ),
            blocksDict = listOf(
                BlocksDto("BLK001", "Block 1", "BLK1", "curatorName"),
                BlocksDto("BLK002", "Block 2", "BLK2", "curatorName")
            ),
            processData = Pair(
                emptyList(),
                listOf(
                    ProcessesMappingDto("PROC001", "DPSS001", 1),
                    ProcessesMappingDto("PROC002", "DPSS002", 2)
                )
            ),
            terBanksRefs = mutableListOf(
                TerBank().also {
                    it.id = 1
                    it.name = "Bank 1"
                    it.shortName = "TB"
                }
            )
        )

        transactionStatus = mockk(relaxed = true)
        val callbackSlot = slot<TransactionCallback<*>>()

        every { transactionTemplate.execute(capture(callbackSlot)) } answers {
            callbackSlot.captured.doInTransaction(transactionStatus)
        }

        setupCommonMocks()
    }

    @Test
    fun `processAgentSheet - with full agent data`() {
        val agentId = "AGENT001"
        setupMocksForFullDataScenario(agentId)

        agentSheetService.processAgentSheet(context, mockSheet, 1L)

        verify(exactly = 1) { dictionariesProvider.load() }
        verify(exactly = 1) { aIAgentRepository.findAllByAgentIdIn(setOf(agentId)) }
        verify(exactly = 1) {
            contactRepository.findAllByEmailIn(
                setOf("test@sberbank.ru", "test2@sber.ru")
            )
        }
        verify(exactly = 1) {
            aiAgentQualityGateService.createOrUpdateByAgent(match { it == listOf(1L) })
        }
        verify(exactly = 1) {
            appliedMetricRepository.saveAll(any<Iterable<AppliedMetricsEntity>>())
        }
        verify(exactly = 1) { processRepository.refreshStatusesFromAgents() }
        verify(exactly = 1) {
            fileUploadRepository.save(match { it.status == FileUploadStatus.SUCCESS })
        }

        // initial clear after dictionary snapshot + clear after the only batch
        verify(exactly = 2) { entityManager.clear() }

        // flush after batch + final flush
        verify(exactly = 2) { entityManager.flush() }
    }

    @Test
    fun `processAgentSheet - with empty data`() {
        every { mockSheet.iterator() } answers { mockRowIteratorWithEmptyData() }

        val exception = assertThrows<AiResponseException> {
            agentSheetService.processAgentSheet(context, mockSheet, 1L)
        }

        assertEquals(AI_UPLOAD_EMPTY_FILE, exception.errorCode)
        assertEquals(HttpStatus.BAD_REQUEST, exception.status)

        verify(exactly = 1) { transactionStatus.setRollbackOnly() }
        verify(exactly = 0) { fileUploadRepository.save(any()) }
        verify(exactly = 0) { processRepository.refreshStatusesFromAgents() }
    }

    @Test
    fun `processAgentSheet - with throw exception when wrong ter bank`() {
        val agentId = "AGENT001"
        setupMocksForWrongTerBankScenario(agentId)

        val exception = assertThrows<AiResponseException> {
            agentSheetService.processAgentSheet(context, mockSheet, 1L)
        }

        assertEquals(AI_UPLOAD_UNKNOWN_TER_BANK, exception.errorCode)
        assertEquals(HttpStatus.BAD_REQUEST, exception.status)
        verify(exactly = 1) { transactionStatus.setRollbackOnly() }
    }

    @Test
    fun `processAgentSheet - with throw exception when wrong status`() {
        val agentId = "AGENT001"
        setupMocksForWrongStatusScenario(agentId)

        val exception = assertThrows<AiResponseException> {
            agentSheetService.processAgentSheet(context, mockSheet, 1L)
        }

        assertEquals(AI_UPLOAD_UNKNOWN_AGENT_STATUS, exception.errorCode)
        assertEquals(HttpStatus.BAD_REQUEST, exception.status)
        verify(exactly = 1) { transactionStatus.setRollbackOnly() }
    }

    @Test
    fun `processAgentSheet - should throw when duplicate agent id in file`() {
        val agentId = "AGENT001"

        every { mockSheet.iterator() } answers {
            mockRowIteratorWithDuplicateAgents(agentId)
        }
        every { messageProvider[AI_UPLOAD_DOUBLES] } returns "Duplicate AI agent id {0}"

        val exception = assertThrows<AiResponseException> {
            agentSheetService.processAgentSheet(context, mockSheet, 1L)
        }

        assertEquals(AI_UPLOAD_DOUBLES, exception.errorCode)
        assertEquals(HttpStatus.BAD_REQUEST, exception.status)
        verify(exactly = 1) { transactionStatus.setRollbackOnly() }
    }

    @Test
    fun `processAgentSheet - should set completedDate for confirm effect when target decision finished`() {
        val agentId = "AGENT-COMPLETE"

        val existingAgent = AIAgentEntity().apply {
            id = 10L
            this.agentId = agentId
            this.agentStatusSla = mutableListOf()
        }

        val savedSlas = mutableListOf<AgentStatusSlaEntity>()

        every { mockSheet.iterator() } answers {
            mockRowIteratorWithFullData(
                agentId = agentId,
                targetDecisionValue = "Завершен",
                terBankValue = "TB",
                includeConfirmEffectDate = true
            )
        }
        every {
            aIAgentRepository.findAllByAgentIdIn(setOf(agentId))
        } returns listOf(existingAgent)
        every {
            agentStatusSlaRepository.findAllByAiAgentIdIn(listOf(10L))
        } returns emptyList()
        every { agentStatusSlaRepository.save(capture(savedSlas)) } answers {
            prepareSavedSla(firstArg())
        }

        agentSheetService.processAgentSheet(context, mockSheet, 1L)

        assertTrue(
            savedSlas.any {
                it.agentStatus?.code == "targetSolution" &&
                    it.completedDate != null
            }
        )
    }

    @Test
    fun `processAgentSheet - should clear completedDate when target decision not finished`() {
        val agentId = "AGENT-IN-PROGRESS"
        val targetStatus = statusEntity(
            id = 5L,
            name = "target solution",
            code = "targetSolution",
            ordering = 5L
        )

        val existingAgent = AIAgentEntity().apply {
            id = 11L
            this.agentId = agentId
            this.agentStatusSla = mutableListOf()
        }

        val existingTargetSla = sla(
            agent = existingAgent,
            status = targetStatus,
            plannedDate = LocalDateTime.now().plusDays(20),
            completedDate = LocalDateTime.now().minusDays(1)
        )

        val savedSlas = mutableListOf<AgentStatusSlaEntity>()

        every { mockSheet.iterator() } answers {
            mockRowIteratorWithFullData(
                agentId = agentId,
                targetDecisionValue = "В работе",
                terBankValue = "TB",
                includeConfirmEffectDate = true
            )
        }
        every {
            aIAgentRepository.findAllByAgentIdIn(setOf(agentId))
        } returns listOf(existingAgent)
        every {
            agentStatusSlaRepository.findAllByAiAgentIdIn(listOf(11L))
        } returns listOf(existingTargetSla)
        every { agentStatusSlaRepository.save(capture(savedSlas)) } answers {
            prepareSavedSla(firstArg())
        }

        agentSheetService.processAgentSheet(context, mockSheet, 1L)

        assertTrue(
            savedSlas.any {
                it.agentStatus?.code == "targetSolution" &&
                    it.completedDate == null
            }
        )
    }

    @Test
    fun `processAgentSheet - should compute deadlineExpired expired`() {
        val agentId = "AGENT-EXPIRED"
        val targetStatus = statusEntity(
            id = 5L,
            name = "target solution",
            code = "targetSolution",
            ordering = 5L
        )

        val existingAgent = AIAgentEntity().apply {
            id = 12L
            this.agentId = agentId
            this.agentStatusSla = mutableListOf()
        }

        val existingSla = sla(
            agent = existingAgent,
            status = targetStatus,
            plannedDate = LocalDateTime.now().minusDays(1)
        )

        every { mockSheet.iterator() } answers {
            mockRowIteratorWithFullData(
                agentId = agentId,
                targetDecisionValue = "В работе",
                terBankValue = "TB",
                includeConfirmEffectDate = false
            )
        }
        every { dateTimeProvider.currentDate() } returns LocalDate.now()
        every {
            aIAgentRepository.findAllByAgentIdIn(setOf(agentId))
        } returns listOf(existingAgent)
        every {
            agentStatusSlaRepository.findAllByAiAgentIdIn(listOf(12L))
        } returns listOf(existingSla)

        agentSheetService.processAgentSheet(context, mockSheet, 1L)

        assertEquals(DeadlineExpiredType.expired, existingAgent.deadlineExpired)
    }

    @Test
    fun `processAgentSheet - should compute deadlineExpired expiration`() {
        val agentId = "AGENT-EXPIRATION"
        val targetStatus = statusEntity(
            id = 5L,
            name = "target solution",
            code = "targetSolution",
            ordering = 5L
        )

        val existingAgent = AIAgentEntity().apply {
            id = 13L
            this.agentId = agentId
            this.agentStatusSla = mutableListOf()
        }

        val existingSla = sla(
            agent = existingAgent,
            status = targetStatus,
            plannedDate = LocalDateTime.now().plusDays(5)
        )

        every { mockSheet.iterator() } answers {
            mockRowIteratorWithFullData(
                agentId = agentId,
                targetDecisionValue = "В работе",
                terBankValue = "TB",
                includeConfirmEffectDate = false
            )
        }
        every { dateTimeProvider.currentDate() } returns LocalDate.now()
        every {
            aIAgentRepository.findAllByAgentIdIn(setOf(agentId))
        } returns listOf(existingAgent)
        every {
            agentStatusSlaRepository.findAllByAiAgentIdIn(listOf(13L))
        } returns listOf(existingSla)

        agentSheetService.processAgentSheet(context, mockSheet, 1L)

        assertEquals(DeadlineExpiredType.expiration, existingAgent.deadlineExpired)
    }

    @Test
    fun `processAgentSheet - should skip blocked agent and save it to blockedAgents`() {
        val agentId = "AGENT-BLOCKED"

        val existingAgent = AIAgentEntity().apply {
            id = 14L
            this.agentId = agentId
            this.agentName = "Blocked agent"
            this.importStatus = "blocked"
            this.agentStatusSla = mutableListOf()
        }

        every { mockSheet.iterator() } answers {
            mockRowIteratorWithFullData(
                agentId = agentId,
                targetDecisionValue = "Завершен",
                terBankValue = "TB",
                includeConfirmEffectDate = true
            )
        }
        every {
            aIAgentRepository.findAllByAgentIdIn(setOf(agentId))
        } returns listOf(existingAgent)

        agentSheetService.processAgentSheet(context, mockSheet, 1L)

        assertTrue(
            fileUploadEntity.blockedAgents?.any {
                it.agentId == "14" && it.agentName == "Blocked agent"
            } == true
        )

        verify(exactly = 0) {
            aiAgentQualityGateService.createOrUpdateByAgent(any())
        }
    }

    @Test
    fun `processAgentSheet - should disable deleted agents by projection`() {
        val processedAgentId = "AGENT001"
        setupMocksForFullDataScenario(processedAgentId)

        every {
            aIAgentRepository.findAllNonPultAgentRefs()
        } returns listOf(
            agentRef(1L, processedAgentId),
            agentRef(99L, "DELETED_AGENT")
        )

        val disabledIds = mutableListOf<Collection<Long>>()
        every { aIAgentRepository.disableByIds(any()) } answers {
            disabledIds += firstArg<Collection<Long>>().toList()
            firstArg<Collection<Long>>().size
        }

        agentSheetService.processAgentSheet(context, mockSheet, 1L)

        assertEquals(listOf(listOf(99L)), disabledIds.map { it.toList() })
    }

    @Test
    fun `processAgentSheet - should not disable agent present in file`() {
        val processedAgentId = "AGENT001"
        setupMocksForFullDataScenario(processedAgentId)

        every {
            aIAgentRepository.findAllNonPultAgentRefs()
        } returns listOf(agentRef(1L, processedAgentId))

        agentSheetService.processAgentSheet(context, mockSheet, 1L)

        verify(exactly = 0) { aIAgentRepository.disableByIds(any()) }
    }

    @Test
    fun `processAgentSheet - should process 250 agents in 3 batches and clear persistence context`() {
        val agentIds = (1..250).map { "AGENT-${it.toString().padStart(3, '0')}" }

        every { mockSheet.iterator() } answers {
            mockRowIteratorWithMinimalAgents(agentIds)
        }

        val requestedBatches = mutableListOf<Set<String>>()
        every {
            aIAgentRepository.findAllByAgentIdIn(any<Collection<String>>())
        } answers {
            requestedBatches += firstArg<Collection<String>>().toSet()
            emptyList()
        }

        agentSheetService.processAgentSheet(context, mockSheet, 1L)

        assertEquals(listOf(100, 100, 50), requestedBatches.map { it.size })

        verify(exactly = 1) { dictionariesProvider.load() }

        // initial clear + 3 clears after batches
        verify(exactly = 4) { entityManager.clear() }

        // 3 batch flushes + final flush
        verify(exactly = 4) { entityManager.flush() }

        verify(exactly = 1) {
            aiAgentQualityGateService.createOrUpdateByAgent(
                match { it.size == 250 }
            )
        }
    }

    @Test
    fun `processAgentSheet - should rollback whole import when second batch fails`() {
        val agentIds = (1..100).map { "AGENT-${it.toString().padStart(3, '0')}" } +
            "AGENT-001"

        every { mockSheet.iterator() } answers {
            mockRowIteratorWithMinimalAgents(agentIds)
        }

        every { messageProvider[AI_UPLOAD_DOUBLES] } returns "Duplicate AI agent id {0}"

        val exception = assertThrows<AiResponseException> {
            agentSheetService.processAgentSheet(context, mockSheet, 1L)
        }

        assertEquals(AI_UPLOAD_DOUBLES, exception.errorCode)

        verify(exactly = 1) { transactionStatus.setRollbackOnly() }

        // first batch was flushed, second one failed before its flush
        verify(exactly = 1) { entityManager.flush() }

        // initial clear + clear after first successful batch
        verify(exactly = 2) { entityManager.clear() }

        verify(exactly = 0) { fileUploadRepository.save(any()) }
    }

    @Test
    fun `processAgentSheet - should split quality gates into chunks of 500`() {
        val agentIds = (1..501).map { "AGENT-${it.toString().padStart(3, '0')}" }

        every { mockSheet.iterator() } answers {
            mockRowIteratorWithMinimalAgents(agentIds)
        }

        val qualityGateChunks = mutableListOf<List<Long>>()
        justRun {
            aiAgentQualityGateService.createOrUpdateByAgent(capture(qualityGateChunks))
        }

        agentSheetService.processAgentSheet(context, mockSheet, 1L)

        assertEquals(listOf(500, 1), qualityGateChunks.map { it.size })

        verify(exactly = 2) {
            aiAgentQualityGateService.removeIrrelevant(any<List<Long>>())
        }
    }

    @Test
    fun `processAgentSheet - should disable non actual duplicate from database`() {
        val agentId = "AGENT-DUPLICATED"

        val oldDuplicate = AIAgentEntity().apply {
            id = 10L
            this.agentId = agentId
            disabled = false
            updated = LocalDateTime.now().minusDays(10)
        }

        val actualAgent = AIAgentEntity().apply {
            id = 11L
            this.agentId = agentId
            disabled = false
            updated = LocalDateTime.now()
        }

        every { mockSheet.iterator() } answers {
            mockRowIteratorWithMinimalAgents(listOf(agentId))
        }
        every {
            aIAgentRepository.findAllByAgentIdIn(setOf(agentId))
        } returns listOf(oldDuplicate, actualAgent)
        every {
            agentStatusSlaRepository.findAllByAiAgentIdIn(listOf(11L))
        } returns emptyList()

        agentSheetService.processAgentSheet(context, mockSheet, 1L)

        assertTrue(oldDuplicate.disabled == true)
        assertFalse(actualAgent.disabled == true)
    }

    @Test
    fun `processAgentSheet - should execute contact cleanup before dictionary loading`() {
        every { mockSheet.iterator() } answers {
            mockRowIteratorWithMinimalAgents(listOf("AGENT001"))
        }

        agentSheetService.processAgentSheet(context, mockSheet, 1L)

        verifyOrder {
            agentContactRepository.deleteAllForImportedNonBlockedAgents()
            dictionariesProvider.load()
            entityManager.clear()
        }
    }

    private fun setupCommonMocks() {
        val activeStatus = statusEntity(1L, "active", "active", 1L)
        val analysisStatus = statusEntity(2L, "analysis", "analysis", 2L)
        val developmentStatus = statusEntity(3L, "development", "development", 3L)
        val pilotStatus = statusEntity(4L, "pilot", "pilot", 4L)
        val targetSolutionStatus = statusEntity(
            5L,
            "target solution",
            "targetSolution",
            5L
        )

        val statuses = listOf(
            activeStatus,
            analysisStatus,
            developmentStatus,
            pilotStatus,
            targetSolutionStatus
        )

        val statusRefs = statuses.map {
            StatusRef(
                id = requireNotNull(it.id),
                code = it.code,
                name = it.name,
                ordering = it.ordering
            )
        }

        val dictionaries = AgentImportDictionaries(
            statusesByName = statusRefs.associateBy {
                it.name!!.trim().lowercase()
            },
            statusesByCode = statusRefs.associateBy {
                it.code!!.trim().lowercase()
            },
            statusesById = statusRefs.associateBy { it.id },
            metricsByFileName = mapOf(
                "metric" to MetricRef(
                    id = 1000L,
                    fileName = "metric"
                )
            ),
            resourcesByName = mapOf(
                "resource1" to ResourceRef(
                    name = "resource1",
                    source = "source1",
                    type = "type1"
                )
            ),
            divisionsByCode = mapOf(
                "div001" to DivisionRef(
                    id = 2000L,
                    code = "DIV001",
                    blockId = null
                )
            ),
            blocksByShortName = mapOf(
                "block" to BlockRef(
                    id = 3000L,
                    shortName = "block"
                )
            ),
            processesByShortName = mapOf(
                "process shortName" to ProcessRef(
                    id = 4000L,
                    shortName = "process shortName"
                )
            ),
            programsByFileName = emptyMap(),
            initiativeTypesByCode = emptyMap(),
            platformsByName = mapOf(
                "platform1" to PlatformRef(
                    id = 5000L,
                    name = "platform1"
                )
            )
        )

        every { dictionariesProvider.load() } returns dictionaries

        val statusById = statuses.associateBy { requireNotNull(it.id) }

        every {
            entityManager.getReference(StatusEntity::class.java, any())
        } answers {
            statusById[secondArg<Long>()]
                ?: error("Unknown status id ${secondArg<Long>()}")
        }

        every {
            entityManager.getReference(DivisionEntity::class.java, any())
        } returns mockk(relaxed = true)

        every {
            entityManager.getReference(BlockEntity::class.java, any())
        } returns mockk(relaxed = true)

        every {
            entityManager.getReference(ProcessEntity::class.java, any())
        } returns mockk(relaxed = true)

        every {
            entityManager.getReference(ProgramEntity::class.java, any())
        } returns mockk(relaxed = true)

        every {
            entityManager.getReference(InitiativeTypeEntity::class.java, any())
        } returns mockk(relaxed = true)

        every {
            entityManager.getReference(MetricEntity::class.java, any())
        } returns mockk(relaxed = true)

        every {
            entityManager.getReference(PlatformEntity::class.java, any())
        } returns mockk(relaxed = true)

        every {
            agentContactRepository.deleteAllForImportedNonBlockedAgents()
        } returns 0

        every {
            aIAgentRepository.findAllNonPultAgentRefs()
        } returns emptyList()

        every {
            aIAgentRepository.disableByIds(any())
        } returns 0

        every {
            processRepository.refreshStatusesFromAgents()
        } returns 0

        every {
            aIAgentRepository.findAllByAgentIdIn(any<Collection<String>>())
        } returns emptyList()

        every {
            aIAgentRepository.save(any())
        } answers {
            firstArg<AIAgentEntity>().also {
                if (it.id == null) {
                    it.id = nextAgentId++
                }
            }
        }

        every {
            agentStatusSlaRepository.findAllByAiAgentIdIn(any<Collection<Long>>())
        } returns emptyList()

        every {
            agentStatusSlaRepository.save(any())
        } answers {
            prepareSavedSla(firstArg())
        }

        justRun {
            appliedMetricRepository.deleteAllByAiAgentId(any())
        }

        every {
            appliedMetricRepository.saveAll(any<Iterable<AppliedMetricsEntity>>())
        } answers {
            firstArg<Iterable<AppliedMetricsEntity>>().toList()
        }

        every {
            contactRepository.findAllByEmailIn(any<Collection<String>>())
        } returns emptyList()

        every {
            contactRepository.save(any())
        } answers {
            firstArg()
        }

        justRun {
            aiAgentQualityGateService.createOrUpdateByAgent(any())
        }

        justRun {
            aiAgentQualityGateService.removeIrrelevant(any<List<Long>>())
        }

        every {
            fileUploadRepository.findById(1L)
        } returns java.util.Optional.of(fileUploadEntity)

        every {
            fileUploadRepository.save(any())
        } answers {
            firstArg()
        }

        every { dateTimeProvider.currentDate() } returns LocalDate.now()

        every {
            messageProvider[AI_UPLOAD_EMPTY_FILE]
        } returns "File is empty or contains only header"

        every {
            messageProvider[AI_UPLOAD_DOUBLES]
        } returns "Duplicate AI agent id {0}"

        every {
            messageProvider[AI_UPLOAD_UNKNOWN_TER_BANK, any(), any()]
        } returns "Unknown territory bank"

        every {
            messageProvider[AI_UPLOAD_UNKNOWN_AGENT_STATUS, any(), any()]
        } returns "Unknown agent status"
    }

    private fun setupMocksForFullDataScenario(agentId: String) {
        val targetStatus = statusEntity(
            id = 5L,
            name = "target solution",
            code = "targetSolution",
            ordering = 5L
        )

        every { mockSheet.iterator() } answers {
            mockRowIteratorWithFullData(
                agentId = agentId,
                targetDecisionValue = "Завершен",
                terBankValue = "TB",
                includeConfirmEffectDate = true
            )
        }

        val existingAgent = AIAgentEntity().apply {
            id = 1L
            this.agentId = agentId
            this.agentStatusSla = mutableListOf()
        }

        every {
            aIAgentRepository.findAllByAgentIdIn(setOf(agentId))
        } returns listOf(existingAgent)

        val existingSla = sla(
            agent = existingAgent,
            status = targetStatus,
            plannedDate = LocalDateTime.now().minusDays(10)
        )

        every {
            agentStatusSlaRepository.findAllByAiAgentIdIn(listOf(1L))
        } returns listOf(existingSla)
    }

    private fun setupMocksForWrongTerBankScenario(agentId: String) {
        every { mockSheet.iterator() } answers {
            mockRowIteratorWithWrongTBData(agentId)
        }

        val existingAgent = AIAgentEntity().apply {
            id = 1L
            this.agentId = agentId
            this.agentStatusSla = mutableListOf()
        }

        every {
            aIAgentRepository.findAllByAgentIdIn(setOf(agentId))
        } returns listOf(existingAgent)
    }

    private fun setupMocksForWrongStatusScenario(agentId: String) {
        every { mockSheet.iterator() } answers {
            mockRowIteratorWithWrongStatus(agentId)
        }
    }

    private fun prepareSavedSla(sla: AgentStatusSlaEntity): AgentStatusSlaEntity {
        sla.primaryKey = AIAgentStatusPK().apply {
            aiAgentId = sla.aiAgent?.id
            agentStatusId = sla.agentStatus?.id
        }
        return sla
    }

    private fun sla(
        agent: AIAgentEntity,
        status: StatusEntity,
        plannedDate: LocalDateTime?,
        completedDate: LocalDateTime? = null
    ): AgentStatusSlaEntity =
        AgentStatusSlaEntity().apply {
            primaryKey = AIAgentStatusPK().apply {
                aiAgentId = agent.id
                agentStatusId = status.id
            }
            aiAgent = agent
            agentStatus = status
            this.plannedDate = plannedDate
            this.completedDate = completedDate
        }

    private fun statusEntity(
        id: Long,
        name: String,
        code: String,
        ordering: Long
    ): StatusEntity =
        StatusEntity().apply {
            this.id = id
            this.name = name
            this.code = code
            this.ordering = ordering
        }

    private fun agentRef(
        id: Long,
        agentId: String
    ): AgentImportRefProjection {
        val projection = mockk<AgentImportRefProjection>()
        every { projection.id } returns id
        every { projection.agentId } returns agentId
        return projection
    }

    private fun mockRowIterator(rows: List<Row>): MutableIterator<Row> {
        return object : MutableIterator<Row> {
            private var index = 0

            override fun hasNext(): Boolean = index < rows.size

            override fun next(): Row = rows[index++]

            override fun remove() = Unit
        }
    }

    private fun mockRowIteratorWithEmptyData(): MutableIterator<Row> {
        val secondRowHeaders = mapOf(
            AGENT_ID to 0,
            AGENT_NAME to 1,
            AGENT_STATUS to 2,
            DIVISION to 3,
            CUSTOMER_CONTACT to 4,
            DEVELOPER_CONTACT to 5,
            AGENT_EFFECT_OPTIMIZATION to 6,
            AGENT_EFFECT_REVENUE to 7,
            "Индивидуальная метрика" to 8,
            "metric" to 9
        )

        val firstHeaderRow = createHeaderRow(
            mapOf(
                METRICS_CELL_INDEX to 8,
                FRONTAL_CELL_INDEX to 10,
                RESOURCES_CELL_INDEX to 12
            )
        )
        val secondHeaderRow = createHeaderRow(secondRowHeaders)

        return mockRowIterator(listOf(firstHeaderRow, secondHeaderRow))
    }

    private fun mockRowIteratorWithDuplicateAgents(
        agentId: String
    ): MutableIterator<Row> {
        val secondRowHeaders = mapOf(
            AGENT_ID to 0,
            AGENT_NAME to 1,
            AGENT_STATUS to 2,
            DIVISION to 3
        )

        val firstHeaderRow = createHeaderRow(
            mapOf(
                METRICS_CELL_INDEX to 10,
                FRONTAL_CELL_INDEX to 12,
                RESOURCES_CELL_INDEX to 14
            )
        )
        val secondHeaderRow = createHeaderRow(secondRowHeaders)

        val divisionShortName = context.divisionsDict.first().shortName

        val row1 = createDataRow(
            mapOf(
                0 to mockStringCell(agentId, 0),
                1 to mockStringCell("agent1", 1),
                2 to mockStringCell("active", 2),
                3 to mockStringCell(divisionShortName, 3)
            ),
            3
        )

        val row2 = createDataRow(
            mapOf(
                0 to mockStringCell(agentId, 0),
                1 to mockStringCell("agent2", 1),
                2 to mockStringCell("active", 2),
                3 to mockStringCell(divisionShortName, 3)
            ),
            4
        )

        return mockRowIterator(
            listOf(firstHeaderRow, secondHeaderRow, row1, row2)
        )
    }

    private fun mockRowIteratorWithMinimalAgents(
        agentIds: List<String>
    ): MutableIterator<Row> {
        val secondRowHeaders = mapOf(
            AGENT_ID to 0,
            AGENT_NAME to 1,
            AGENT_STATUS to 2,
            DIVISION to 3
        )

        val firstHeaderRow = createHeaderRow(
            mapOf(
                METRICS_CELL_INDEX to 10,
                FRONTAL_CELL_INDEX to 12,
                RESOURCES_CELL_INDEX to 14
            )
        )
        val secondHeaderRow = createHeaderRow(secondRowHeaders)

        val divisionShortName = context.divisionsDict.first().shortName

        val rows = agentIds.mapIndexed { index, agentId ->
            createDataRow(
                mapOf(
                    0 to mockStringCell(agentId, 0),
                    1 to mockStringCell("agent $agentId", 1),
                    2 to mockStringCell("active", 2),
                    3 to mockStringCell(divisionShortName, 3)
                ),
                index + 3
            )
        }

        return mockRowIterator(
            listOf(firstHeaderRow, secondHeaderRow) + rows
        )
    }

    private fun mockRowIteratorWithFullData(
        agentId: String,
        targetDecisionValue: String,
        terBankValue: String,
        includeConfirmEffectDate: Boolean
    ): MutableIterator<Row> {
        val secondRowHeaders = mapOf(
            AGENT_ID to 0,
            AGENT_NAME to 1,
            AGENT_STATUS to 2,
            DIVISION to 3,
            CUSTOMER_CONTACT to 4,
            DEVELOPER_CONTACT to 5,
            AGENT_EFFECT_OPTIMIZATION to 6,
            AGENT_EFFECT_REVENUE to 7,
            PROCESS_CODE to 8,
            POC to 9,
            MVP to 10,
            TARGET_DECISION to 11,
            CONFIRM_EFFECT to 12,
            TER_BANK to 13,
            "Индивидуальная метрика" to 14,
            "metric" to 15,
            "platform1" to 16,
            "platform2" to 17,
            "resource1" to 18,
            "resource2" to 19,
            "resource3" to 20,
            "resource4" to 21
        )

        val firstHeaderRow = createHeaderRow(
            mapOf(
                METRICS_CELL_INDEX to 14,
                FRONTAL_CELL_INDEX to 16,
                RESOURCES_CELL_INDEX to 18
            )
        )
        val secondHeaderRow = createHeaderRow(secondRowHeaders)

        val cells = mutableMapOf<Int, Cell>(
            0 to mockStringCell(agentId, 0),
            1 to mockStringCell("agent name", 1),
            2 to mockStringCell("active", 2),
            3 to mockStringCell(context.divisionsDict.first().shortName, 3),
            4 to mockStringCell("Иванов,test@sberbank.ru", 4),
            5 to mockStringCell("Петров,test2@sber.ru", 5),
            6 to mockNumberCell(123.0, 6),
            7 to mockNumberCell(456.0, 7),
            8 to mockStringCell("process shortName", 8),
            9 to mockDateCell(9),
            10 to mockDateCell(10),
            11 to mockStringCell(targetDecisionValue, 11),
            13 to mockStringCell(terBankValue, 13),
            15 to mockStringCell("12.34", 15),
            16 to mockStringCell("Внедрен", 16),
            18 to mockStringCell("10", 18)
        )

        cells[12] = if (includeConfirmEffectDate) {
            mockDateCell(12)
        } else {
            mockStringCell("", 12)
        }

        val dataRow = createDataRow(cells, 3)

        return mockRowIterator(
            listOf(firstHeaderRow, secondHeaderRow, dataRow)
        )
    }

    private fun mockRowIteratorWithWrongTBData(
        agentId: String
    ): MutableIterator<Row> {
        val secondRowHeaders = mapOf(
            AGENT_ID to 0,
            AGENT_NAME to 1,
            TER_BANK to 2,
            DIVISION to 3,
            AGENT_STATUS to 4
        )

        val firstHeaderRow = createHeaderRow(
            mapOf(
                METRICS_CELL_INDEX to 13,
                FRONTAL_CELL_INDEX to 15,
                RESOURCES_CELL_INDEX to 17
            )
        )
        val secondHeaderRow = createHeaderRow(secondRowHeaders)

        val dataRow = createDataRow(
            mapOf(
                0 to mockStringCell(agentId, 0),
                1 to mockStringCell("agent name", 1),
                2 to mockStringCell("TB-2", 2),
                3 to mockStringCell(context.divisionsDict.first().shortName, 3),
                4 to mockStringCell("active", 4)
            ),
            3
        )

        return mockRowIterator(
            listOf(firstHeaderRow, secondHeaderRow, dataRow)
        )
    }

    private fun mockRowIteratorWithWrongStatus(
        agentId: String
    ): MutableIterator<Row> {
        val secondRowHeaders = mapOf(
            AGENT_ID to 0,
            AGENT_NAME to 1,
            AGENT_STATUS to 2
        )

        val firstHeaderRow = createHeaderRow(
            mapOf(
                METRICS_CELL_INDEX to 13,
                FRONTAL_CELL_INDEX to 15,
                RESOURCES_CELL_INDEX to 17
            )
        )
        val secondHeaderRow = createHeaderRow(secondRowHeaders)

        val dataRow = createDataRow(
            mapOf(
                0 to mockStringCell(agentId, 0),
                1 to mockStringCell("agent name", 1),
                2 to mockStringCell("wrong", 2)
            ),
            3
        )

        return mockRowIterator(
            listOf(firstHeaderRow, secondHeaderRow, dataRow)
        )
    }

    private fun createDataRow(
        cells: Map<Int, Cell>,
        rowNum: Int
    ): Row {
        val row = mockk<Row>()

        every { row.getCell(any()) } answers {
            cells[firstArg<Int>()]
        }
        every { row.rowNum } returns rowNum
        every { row.iterator() } answers {
            cells.values
                .sortedBy { it.columnIndex }
                .iterator() as MutableIterator<Cell>
        }

        return row
    }

    private fun mockStringCell(
        value: String,
        index: Int
    ): Cell {
        val cell = mockk<Cell>()

        every { cell.cellType } returns CellType.STRING
        every { cell.stringCellValue } returns value
        every { cell.columnIndex } returns index
        every { cell.rowIndex } returns 2

        return cell
    }

    private fun mockNumberCell(
        value: Double,
        index: Int
    ): Cell {
        val cell = mockk<Cell>()

        every { cell.cellType } returns CellType.NUMERIC
        every { cell.cellStyle } returns mockk {
            every { dataFormatString } returns "12"
            every { dataFormat } returns 12
        }
        every { cell.numericCellValue } returns value
        every { cell.stringCellValue } returns value.toString()
        every { cell.columnIndex } returns index
        every { cell.rowIndex } returns 2

        return cell
    }

    private fun mockDateCell(index: Int): Cell {
        val cell = mockk<Cell>()

        every { cell.cellType } returns CellType.NUMERIC
        every { cell.cellStyle } returns mockk {
            every { dataFormatString } returns "14"
            every { dataFormat } returns 14
        }
        every { cell.dateCellValue } returns Date()
        every { cell.numericCellValue } returns 2.0
        every { cell.stringCellValue } returns "2.0"
        every { cell.columnIndex } returns index
        every { cell.rowIndex } returns 2

        return cell
    }

    private fun createHeaderRow(
        headers: Map<String, Int>
    ): Row {
        val headerRow = mockk<Row>()

        val cells = headers.map { (key, index) ->
            val cell = mockk<Cell>()
            every { cell.cellType } returns CellType.STRING
            every { cell.stringCellValue } returns key
            every { cell.columnIndex } returns index
            every { headerRow.getCell(index) } returns cell
            cell
        }.toMutableList()

        every { headerRow.iterator() } answers {
            cells.iterator()
        }

        return headerRow
    }
}

```
