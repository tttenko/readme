```java

@DisplayName("InitiativeRegistryExportService tests")
class InitiativeRegistryExportServiceTest {

    private lateinit var terBankService: TerBankService
    private lateinit var aiAgentRepository: AIAgentRepository
    private lateinit var statusRepository: StatusRepository
    private lateinit var agentStatusSlaRepository: AgentStatusSlaRepository
    private lateinit var authFeignClient: PrmAuthFeignClient
    private lateinit var financialEffectRepository: InitiativeFinancialEffectRepository
    private lateinit var service: InitiativeRegistryExportService

    private val objectMapper = ObjectMapper()

    @BeforeEach
    fun setUp() {
        terBankService = mockk()
        aiAgentRepository = mockk()
        statusRepository = mockk()
        agentStatusSlaRepository = mockk()
        authFeignClient = mockk()
        financialEffectRepository = mockk()

        service = InitiativeRegistryExportService(
            mapper = InitiativeRegistryMapper(),
            terBankService = terBankService,
            aiAgentRepository = aiAgentRepository,
            statusRepository = statusRepository,
            agentStatusSlaRepository = agentStatusSlaRepository,
            authFeignClient = authFeignClient,
            financialEffectRepository = financialEffectRepository
        )

        ReflectionUtils.setField(service, "portalUrl", "https://pult.sber.ru/")
    }

    private fun status(code: String, ordering: Long) = StatusEntity().also {
        it.id = ordering
        it.code = code
        it.name = code
        it.ordering = ordering
    }

    private fun sla(statusCode: String, plannedDate: LocalDateTime? = null, initiativeId: Long? = 1L) =
        AgentStatusSlaEntity().also {
            it.primaryKey.aiAgentId = initiativeId
            it.agentStatus = status(statusCode, 1L)
            it.plannedDate = plannedDate
        }

    private fun agent(id: Long) = AIAgentEntity().also {
        it.id = id
        it.agentId = "AI-$id"
        it.agentName = "Инициатива $id"
    }

    private fun financialEffect(
        id: Long,
        initiativeId: Long,
        effectYear: Int = 2026,
        plan: String = "1000",
        fact: String? = "1200",
        targetMonth: LocalDate = YearMonth.of(2026, 3).atDay(1),
        code: String = "revenue",
        typeName: String = "Выручка",
        unit: String = "руб.",
        approvalLinks: List<String> = listOf("https://jira.example.com/PRM-1"),
    ): InitiativeFinancialEffectEntity {
        val type = FinancialEffectTypeEntity(
            code = code,
            name = typeName,
            unit = unit,
            moreIsBetter = true,
        )
        return InitiativeFinancialEffectEntity(
            initiative = AIAgentEntity().apply { this.id = initiativeId },
            effectYear = effectYear,
            type = type,
            planValue = BigDecimal(plan),
            factValue = fact?.let { BigDecimal(it) },
            targetMonth = targetMonth,
            approvalLinks = objectMapper.valueToTree(approvalLinks),
            disabled = false,
            createdBy = 1L,
        ).apply {
            this.id = id
            this.created = LocalDateTime.of(effectYear, 1, 1, 0, 0)
        }
    }

    private fun mockExportDependencies(agents: List<AIAgentEntity> = listOf(agent(1L))) {
        every { terBankService.list() } returns emptyList()
        every { aiAgentRepository.findAll() } returns agents
        every { statusRepository.findAllActive() } returns emptyList()
        every { agentStatusSlaRepository.findAllByAiAgentIdIn(any()) } returns emptyList()
        every { financialEffectRepository.findActiveByInitiativeIds(any()) } returns emptyList()
    }

    @Test
    fun `should use findAll including disabled and load sla in bulk`() {
        // given
        val activeAgent = AIAgentEntity().also {
            it.id = 1L
            it.agentId = "AI-1"
            it.agentStatus = status("pilot", 3L)
            it.disabled = false
        }

        val disabledAgent = AIAgentEntity().also {
            it.id = 2L
            it.agentId = "AI-2"
            it.agentStatus = status("development", 2L)
            it.disabled = true
        }

        every { terBankService.list() } returns emptyList()
        every { aiAgentRepository.findAll() } returns listOf(activeAgent, disabledAgent)
        every { statusRepository.findAllActive() } returns listOf(
            status("analysis", 1L),
            status("development", 2L),
            status("pilot", 3L),
            status("targetSolution", 4L),
        )
        every { agentStatusSlaRepository.findAllByAiAgentIdIn(any()) } returns
                listOf(sla("development", LocalDateTime.of(2026, 9, 1, 10, 0)))
        every { financialEffectRepository.findActiveByInitiativeIds(any()) } returns emptyList()

        // when
        val response = service.downloadRegistryExcel()

        // then
        verify(exactly = 1) { aiAgentRepository.findAll() }
        verify(exactly = 1) { agentStatusSlaRepository.findAllByAiAgentIdIn(listOf(1L, 2L)) }
        verify(exactly = 0) { agentStatusSlaRepository.findAllByAiAgentId(any()) }
        verify(exactly = 1) { financialEffectRepository.findActiveByInitiativeIds(setOf(1L, 2L)) }

        assertEquals(HttpStatus.OK, response.statusCode)
        assertEquals(InitiativeRegistryExportService.XLSX_MEDIA_TYPE, response.headers.contentType)
    }

    @Test
    fun `should set exact content disposition filename`() {
        // given
        val agent = AIAgentEntity().also {
            it.id = 1L
            it.agentId = "AI-1"
        }

        mockExportDependencies(listOf(agent))

        // when
        val response = service.downloadRegistryExcel()

        // then
        val disposition = ContentDisposition.parse(response.headers.getFirst(HttpHeaders.CONTENT_DISPOSITION)!!)

        assertEquals(InitiativeRegistryExportService.FILE_NAME, disposition.filename)
    }

    @Test
    fun `should produce 27 columns in the required order`() {
        // given
        val agent = AIAgentEntity().also {
            it.id = 1L
            it.agentId = "AI-1"
        }

        mockExportDependencies(listOf(agent))

        // when
        val response = service.downloadRegistryExcel()

        // then
        XSSFWorkbook(response.body!!.inputStream).use { workbook ->
            val headerRow = workbook.getSheetAt(0).getRow(0)

            assertEquals(27, headerRow.physicalNumberOfCells)

            val expectedHeaders = listOf(
                "ID AI-агента",
                "Блок",
                "Трайб",
                "ТБ",
                "Наименование",
                "Описание",
                "Проблема, которую решает",
                "Текущий статус",
                "Ссылка JIRA",
                "CROSSGOAL",
                "GIGAUSAGE",
                "Тип инициативы",
                "Аудитория",
                "Программа AI-трансформации",
                "Поверхности",
                "Архив",
                "Ссылка на инициативу в Пульт",
                "Email контакта",
                "Статус этапа — Концепция",
                "Дедлайн этапа — Концепция",
                "Статус этапа — PoC",
                "Дедлайн этапа — PoC",
                "Статус этапа — MVP",
                "Дедлайн этапа — MVP",
                "Статус этапа — Целевое решение",
                "Дедлайн этапа — Целевое решение",
                "КЭ модуля"
            )

            val headers = (0 until 27).map { headerRow.getCell(it).stringCellValue }

            assertEquals(expectedHeaders, headers)
        }
    }

    @Test
    fun `should set active excel hyperlink for initiative url column`() {
        // given
        val agent = AIAgentEntity().also {
            it.id = 100L
            it.agentId = "AI-100"
            it.disabled = false
        }

        mockExportDependencies(listOf(agent))


        // when
        val response = service.downloadRegistryExcel()

        // then
        XSSFWorkbook(response.body!!.inputStream).use { workbook ->
            val cell = workbook.getSheetAt(0).getRow(1).getCell(16)

            assertNotNull(cell.hyperlink)
            assertEquals("https://pult.sber.ru/ai/initiatives/100", cell.hyperlink.address)
            assertEquals(cell.hyperlink.address, cell.stringCellValue)
        }
    }

    @Test
    fun `should produce xlsx media type`() {
        // given
        val agent = AIAgentEntity().also {
            it.id = 1L
            it.agentId = "AI-1"
        }

        mockExportDependencies(listOf(agent))

        // when
        val response = service.downloadRegistryExcel()

        // then
        assertEquals(
            "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
            response.headers.contentType.toString(),
        )
    }

    @Test
    fun `portal base url without trailing slash should produce correct initiative url`() {
        // given
        val agent = AIAgentEntity().also {
            it.id = 100L
            it.agentId = "AI-100"
        }

        mockExportDependencies(listOf(agent))
        ReflectionUtils.setField(service, "portalUrl", "https://pult.sber.ru")

        // when
        val response = service.downloadRegistryExcel()

        // then
        XSSFWorkbook(response.body!!.inputStream).use { workbook ->
            val cell = workbook.getSheetAt(0).getRow(1).getCell(16)

            assertEquals("https://pult.sber.ru/ai/initiatives/100", cell.stringCellValue)
            assertEquals("https://pult.sber.ru/ai/initiatives/100", cell.hyperlink.address)
        }
    }

    @Test
    fun `valid portal base url produces export`() {
        // given
        val agent = AIAgentEntity().also {
            it.id = 1L
            it.agentId = "AI-1"
        }

        mockExportDependencies(listOf(agent))

        // when
        val response = service.downloadRegistryExcel()

        // then
        assertEquals(HttpStatus.OK, response.statusCode)
    }

    @Test
    fun `should produce financial effect sheet with exact headers`() {
        // given
        mockExportDependencies()

        // when
        val response = service.downloadRegistryExcel()

        // then
        XSSFWorkbook(response.body!!.inputStream).use { workbook ->
            val sheet = workbook.getSheetAt(1)
            val headerRow = sheet.getRow(0)

            assertEquals("Финансовый эффект", sheet.sheetName)
            assertEquals(8, headerRow.physicalNumberOfCells)

            val expectedHeaders = listOf(
                "ID AI-агента",
                "Наименование",
                "Тип эффекта",
                "Год планирования",
                "Срок",
                "План",
                "Факт",
                "Ссылки на jira"
            )

            val headers = (0 until 8).map { headerRow.getCell(it).stringCellValue }

            assertEquals(expectedHeaders, headers)
        }
    }

    @Test
    fun `should write financial effect data in one bulk request`() {
        // given
        val agent = agent(7L)
        val effect = financialEffect(
            id = 10L,
            initiativeId = 7L,
            effectYear = 2026,
            plan = "1000.50",
            fact = "1200",
            targetMonth = YearMonth.of(2026, 3).atDay(1),
            approvalLinks = listOf("https://jira.example.com/PRM-1", "https://jira.example.com/PRM-2"),
        )

        mockExportDependencies(listOf(agent))
        every { financialEffectRepository.findActiveByInitiativeIds(setOf(7L)) } returns listOf(effect)

        // when
        val response = service.downloadRegistryExcel()

        // then
        verify(exactly = 1) { financialEffectRepository.findActiveByInitiativeIds(setOf(7L)) }

        XSSFWorkbook(response.body!!.inputStream).use { workbook ->
            val row = workbook.getSheetAt(1).getRow(1)

            assertEquals("AI-7", row.getCell(0).stringCellValue)
            assertEquals("Инициатива 7", row.getCell(1).stringCellValue)
            assertEquals("Выручка", row.getCell(2).stringCellValue)
            assertEquals(2026.0, row.getCell(3).numericCellValue)
            assertEquals("31.03.2026", row.getCell(4).stringCellValue)
            assertEquals(1000.5, row.getCell(5).numericCellValue)
            assertEquals(1200.0, row.getCell(6).numericCellValue)
            assertEquals(
                "https://jira.example.com/PRM-1; https://jira.example.com/PRM-2",
                row.getCell(7).stringCellValue
            )
            assertEquals(8, row.physicalNumberOfCells)
        }
    }

    @Test
    fun `should write financial effect row with absent fact and empty approval links`() {
        // given
        val agent = agent(7L)
        val effect = financialEffect(
            id = 11L,
            initiativeId = 7L,
            effectYear = 2026,
            plan = "500",
            fact = null,
            targetMonth = YearMonth.of(2026, 5).atDay(1),
            approvalLinks = emptyList(),
        )

        mockExportDependencies(listOf(agent))
        every { financialEffectRepository.findActiveByInitiativeIds(any()) } returns listOf(effect)

        // when
        val response = service.downloadRegistryExcel()

        // then
        XSSFWorkbook(response.body!!.inputStream).use { workbook ->
            val row = workbook.getSheetAt(1).getRow(1)

            assertEquals("AI-7", row.getCell(0).stringCellValue)
            assertEquals("Инициатива 7", row.getCell(1).stringCellValue)
            assertEquals("31.05.2026", row.getCell(4).stringCellValue)
            assertEquals(500.0, row.getCell(5).numericCellValue)
            assertEquals(CellType.BLANK, row.getCell(6).cellType)
            assertEquals("", row.getCell(7).stringCellValue)
        }
    }

    @Test
    fun `should export last day of February including leap year`() {
        // given
        val agent = agent(7L)
        val effects = listOf(
            financialEffect(
                id = 20L,
                initiativeId = 7L,
                effectYear = 2027,
                targetMonth = LocalDate.of(2027, 2, 1)
            ),
            financialEffect(
                id = 21L,
                initiativeId = 7L,
                effectYear = 2028,
                targetMonth = LocalDate.of(2028, 2, 1)
            )
        )

        mockExportDependencies(listOf(agent))
        every { financialEffectRepository.findActiveByInitiativeIds(setOf(7L)) } returns effects

        // when
        val response = service.downloadRegistryExcel()

        // then
        XSSFWorkbook(response.body!!.inputStream).use { workbook ->
            val sheet = workbook.getSheetAt(1)
            assertEquals("28.02.2027", sheet.getRow(1).getCell(4).stringCellValue)
            assertEquals("29.02.2028", sheet.getRow(2).getCell(4).stringCellValue)
            assertEquals(2027.0, sheet.getRow(1).getCell(3).numericCellValue)
            assertEquals(2028.0, sheet.getRow(2).getCell(3).numericCellValue)
        }
    }

    @Test
    fun `should export separate rows for multiple effects of same type and initiative`() {
        // given
        val firstAgent = agent(7L)
        val secondAgent = agent(8L)
        val effects = listOf(
            financialEffect(id = 10L, initiativeId = 7L, plan = "100"),
            financialEffect(id = 11L, initiativeId = 7L, plan = "200"),
            financialEffect(id = 12L, initiativeId = 8L, plan = "300")
        )

        mockExportDependencies(listOf(firstAgent, secondAgent))
        every { financialEffectRepository.findActiveByInitiativeIds(setOf(7L, 8L)) } returns effects

        // when
        val response = service.downloadRegistryExcel()

        // then
        verify(exactly = 1) { financialEffectRepository.findActiveByInitiativeIds(setOf(7L, 8L)) }

        XSSFWorkbook(response.body!!.inputStream).use { workbook ->
            val sheet = workbook.getSheetAt(1)

            assertEquals(4, sheet.physicalNumberOfRows)
            assertEquals("AI-7", sheet.getRow(1).getCell(0).stringCellValue)
            assertEquals("AI-7", sheet.getRow(2).getCell(0).stringCellValue)
            assertEquals("AI-8", sheet.getRow(3).getCell(0).stringCellValue)
            assertEquals("Выручка", sheet.getRow(1).getCell(2).stringCellValue)
            assertEquals("Выручка", sheet.getRow(2).getCell(2).stringCellValue)
            assertEquals(100.0, sheet.getRow(1).getCell(5).numericCellValue)
            assertEquals(200.0, sheet.getRow(2).getCell(5).numericCellValue)
            assertEquals(300.0, sheet.getRow(3).getCell(5).numericCellValue)
        }
    }

    @Test
    fun `should not load financial effects when no initiatives are exported`() {
        // given
        mockExportDependencies(emptyList())

        // when
        val response = service.downloadRegistryExcel()

        // then
        verify(exactly = 0) { financialEffectRepository.findActiveByInitiativeIds(any()) }

        XSSFWorkbook(response.body!!.inputStream).use { workbook ->
            assertEquals(2, workbook.numberOfSheets)
            assertNotNull(workbook.getSheetAt(1).getRow(0))
            assertNull(workbook.getSheetAt(1).getRow(1))
        }
    }

    @Test
    fun `should produce headers only when no financial effects`() {
        // given
        mockExportDependencies()

        // when
        val response = service.downloadRegistryExcel()

        // then
        XSSFWorkbook(response.body!!.inputStream).use { workbook ->
            val sheet = workbook.getSheetAt(1)

            assertNotNull(sheet.getRow(0))
            assertNull(sheet.getRow(1))
        }
    }

    @Test
    fun `should resolve contact emails via one bulk auth request`() {
        // given
        val contact = AgentContactEntity(
            agent = AIAgentEntity(),
            userId = 42L,
        )
        val agent = AIAgentEntity().also {
            it.id = 1L
            it.agentId = "AI-1"
            it.agentContact = mutableListOf(contact)
        }

        mockExportDependencies(listOf(agent))
        every { authFeignClient.getUsers(ids = setOf(42L), companyId = null) } returns
                ResponseEntity.ok(listOf(UserAccountDto(id = 42L, email = "ivan@example.com", secondFactorAuth = "")))

        // when
        val response = service.downloadRegistryExcel()

        // then
        verify(exactly = 1) { authFeignClient.getUsers(ids = setOf(42L), companyId = null) }

        XSSFWorkbook(response.body!!.inputStream).use { workbook ->
            val cell = workbook.getSheetAt(0).getRow(1).getCell(17)

            assertEquals("ivan@example.com", cell.stringCellValue)
        }
    }

    @Test
    fun `should skip blank user emails and leave contact email empty`() {
        // given
        val contact = AgentContactEntity(
            agent = AIAgentEntity(),
            userId = 42L,
        )
        val agent = AIAgentEntity().also {
            it.id = 1L
            it.agentId = "AI-1"
            it.agentContact = mutableListOf(contact)
        }

        mockExportDependencies(listOf(agent))
        every { authFeignClient.getUsers(ids = any(), companyId = null) } returns
                ResponseEntity.ok(
                    listOf(
                        UserAccountDto(id = 42L, email = null, secondFactorAuth = ""),
                        UserAccountDto(id = 43L, email = "   ", secondFactorAuth = ""),
                    ),
                )

        // when
        val response = service.downloadRegistryExcel()

        // then
        XSSFWorkbook(response.body!!.inputStream).use { workbook ->
            val cell = workbook.getSheetAt(0).getRow(1).getCell(17)

            assertEquals("", cell.stringCellValue)
        }
    }

    @Test
    fun `should not call auth service when there are no user contacts`() {
        // given
        mockExportDependencies()

        // when
        val response = service.downloadRegistryExcel()

        // then
        verify(exactly = 0) { authFeignClient.getUsers(any(), any()) }
        assertEquals(HttpStatus.OK, response.statusCode)
    }
}
```
