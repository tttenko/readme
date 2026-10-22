```java

/**
 * Проверяет реальный Excel-импорт в Embedded PostgreSQL (Zonky).
 * Мокается только внешний Feign-клиент тербанков; бизнес-сервисы реальные.
 *
 * ВАЖНО: @SpykBean для ProcessRepository используется для контролируемого
 * сбоя в самом конце бизнес-обработки. Метод до этого момента не подменяется.
 */
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

    @SpykBean
    private lateinit var processRepository: ProcessRepository

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
     * Снимок бизнес-данных включает идентификаторы и значения связей.
     * Исключаем только технические временные поля; file_upload намеренно не берём.
     * Сравнение отсортированных JSON устраняет зависимость от порядка SQL-строк.
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

    @Test
    fun `should roll back all business tables when a late import stage fails`() {
        val before = businessSnapshot()
        val agentCountBefore = aiAgentRepository.count()
        val lateStageReached = AtomicBoolean(false)

        // Метод вызывается ПОСЛЕ сохранения агентов, SLA, метрик и QualityGate.
        // Ошибка в нём должна откатить также справочники и предшествующие пачки.
        every { processRepository.refreshStatusesFromAgents() } answers {
            lateStageReached.set(true)
            val agentsVisibleInsideTransaction = jdbcTemplate.queryForObject(
                "SELECT COUNT(*) FROM ai_agent", Long::class.java
            ) ?: 0L
            assertTrue(
                agentsVisibleInsideTransaction > agentCountBefore,
                "Ошибка должна возникнуть ПОСЛЕ реальной записи агентов внутри транзакции"
            )
            throw IllegalStateException(FORCED_ROLLBACK_MESSAGE)
        }

        val failed = waitUntilFinishedAndUnlocked(startImport())
        assertTrue(lateStageReached.get(), "Тест не дошёл до позднего этапа импорта")
        assertEquals(FileUploadStatus.ERROR, failed.status)
        assertTrue(
            failed.errorDescription?.contains(FORCED_ROLLBACK_MESSAGE) == true,
            "Ожидалась искусственная ошибка, получено: ${failed.errorDescription}"
        )
        assertEquals(before, businessSnapshot(), "После ERROR остались частично применённые данные")
        assertTrue(isUploadAllowed(), "После rollback не освободилась блокировка")
    }

    @Test
    fun `should roll back updates to existing agents when late stage fails`() {
        val initial = waitUntilFinishedAndUnlocked(startImport())
        assertEquals(FileUploadStatus.SUCCESS, initial.status, initial.errorDescription)
        val before = businessSnapshot()
        val (modifiedFile, changedAgentId, changedName) = xlsxWithChangedAgentName()
        val changeWasVisibleBeforeRollback = AtomicBoolean(false)

        every { processRepository.refreshStatusesFromAgents() } answers {
            val nameInsideTransaction = jdbcTemplate.queryForObject(
                "SELECT agent_name FROM ai_agent WHERE agent_id = ? ORDER BY disabled ASC, id DESC LIMIT 1",
                String::class.java, changedAgentId
            )
            changeWasVisibleBeforeRollback.set(nameInsideTransaction == changedName)
            throw IllegalStateException(FORCED_ROLLBACK_MESSAGE)
        }

        val failed = waitUntilFinishedAndUnlocked(startImport(modifiedFile))
        assertEquals(FileUploadStatus.ERROR, failed.status)
        assertTrue(changeWasVisibleBeforeRollback.get(), "Обновление агента не дошло до БД")
        assertEquals(before, businessSnapshot(), "Изменения существующих данных не откатились")
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
