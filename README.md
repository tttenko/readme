```java

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
        // Контакты создаются из модифицированного XLSX; clearTables удаляет
        // agent_contact, но может не удалять самостоятельные записи contact.
        jdbcTemplate.update(
            "DELETE FROM contact WHERE email LIKE 'excel-diff-test-%@sber.ru'"
        )

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

    /**
     * В исходном agents.xlsx могут быть заполнены только обязательные поля.
     * Тесты контактов и метрик НЕ имеют права предполагать, что файл уже
     * содержит их: добавляем данные в копию XLSX перед реальным импортом.
     */
    private data class ImportedAgentRow(val agentId: String, val row: Row)

    private fun findImportedAgentRow(sheet: Sheet, requireNonPult: Boolean): ImportedAgentRow {
        val header = sheet.getRow(1) ?: error("В agents.xlsx отсутствует строка заголовков")
        val idColumn = findHeaderColumn(header, AGENT_ID)
        val formatter = DataFormatter()

        for (rowIndex in 2..sheet.lastRowNum) {
            val row = sheet.getRow(rowIndex) ?: continue
            val agentId = formatter.formatCellValue(row.getCell(idColumn)).trim()
            if (agentId.isBlank() || (requireNonPult && agentId.contains("PULT"))) continue

            val importedAgent = aiAgentRepository.findFirstByAgentId(agentId) ?: continue
            if (importedAgent.importStatus == "blocked" || importedAgent.disabled == true) continue
            return ImportedAgentRow(agentId, row)
        }

        if (requireNonPult) {
            // Фикстура может содержать только PULT-агентов. Берём валидную
            // строку и в тестовой копии XLSX создаём новый non-PULT agentId.
            val originalRow = findImportedAgentRow(sheet, requireNonPult = false).row
            val idColumn = findHeaderColumn(header, AGENT_ID)
            val syntheticAgentId = "excel-diff-test-agent"
            originalRow.getCell(idColumn, Row.MissingCellPolicy.CREATE_NULL_AS_BLANK)
                .setCellValue(syntheticAgentId)
            return ImportedAgentRow(syntheticAgentId, originalRow)
        }
        error("В тестовом XLSX нет ранее импортированного активного агента")
    }

    private fun findHeaderColumn(header: Row, name: String): Int {
        val formatter = DataFormatter()
        return (0 until header.lastCellNum.toInt())
            .firstOrNull { column ->
                formatter.formatCellValue(header.getCell(column)).trim() == name
            }
            ?: error("Отсутствует колонка '$name' в тестовом XLSX")
    }

    private fun findOrCreateContactHeader(sheet: Sheet, name: String): Int {
        val header = sheet.getRow(1) ?: error("Отсутствует строка заголовков")
        val formatter = DataFormatter()
        val existing = (0 until header.lastCellNum.toInt()).firstOrNull { column ->
            formatter.formatCellValue(header.getCell(column)).trim() == name
        }
        if (existing != null) return existing

        // Тестовый шаблон может не содержать опциональные колонки контактов.
        // Добавляем их в конец таблицы, вне диапазонов метрик/ресурсов/платформ.
        val lastUsedColumn = sheet.asSequence()
            .maxOfOrNull { row -> row.lastCellNum.toInt() } ?: 0
        val newColumn = lastUsedColumn.coerceAtLeast(0) + 2
        header.createCell(newColumn).setCellValue(name)
        return newColumn
    }

    private fun editFixture(
        requireNonPult: Boolean = false,
        transform: (Sheet, Row) -> Unit
    ): Pair<MockMultipartFile, String> {
        XSSFWorkbook(ByteArrayInputStream(Files.readAllBytes(Paths.get(FIXTURE_PATH)))).use { workbook ->
            val sheet = workbook.getSheet(AGENTS_SHEET_NAME)
                ?: error("В тестовом XLSX отсутствует лист $AGENTS_SHEET_NAME")
            val chosenAgent = findImportedAgentRow(sheet, requireNonPult)
            transform(sheet, chosenAgent.row)
            val bytes = ByteArrayOutputStream().use { output ->
                workbook.write(output)
                output.toByteArray()
            }
            return multipart(bytes) to chosenAgent.agentId
        }
    }

    private val customerTestContact = "excel-diff-test-customer@sber.ru"
    private val developerTestContact = "excel-diff-test-developer@sber.ru"

    /**
     * Контакты становятся частью входного Excel, а не добавляются напрямую в БД.
     * Можно независимо оставить или убрать каждый контакт для проверки diff.
     */
    private fun xlsxWithContacts(
        includeCustomer: Boolean = true,
        includeDeveloper: Boolean = true
    ): Pair<MockMultipartFile, String> = editFixture(requireNonPult = true) { sheet, row ->
        val customerColumn = findOrCreateContactHeader(sheet, CUSTOMER_CONTACT)
        val developerColumn = findOrCreateContactHeader(sheet, DEVELOPER_CONTACT)

        row.getCell(customerColumn, Row.MissingCellPolicy.CREATE_NULL_AS_BLANK)
            .setCellValue(
                if (includeCustomer) "Integration Customer,$customerTestContact" else ""
            )
        row.getCell(developerColumn, Row.MissingCellPolicy.CREATE_NULL_AS_BLANK)
            .setCellValue(
                if (includeDeveloper) "Integration Developer,$developerTestContact" else ""
            )
    }

    private data class MetricFixture(
        val file: MockMultipartFile,
        val agentId: String,
        val metricId: Long,
        val expectedValue: BigDecimal
    )

    /**
     * Метрики из справочника Liquibase уже существуют, а в исходной fixture
     * их значения отсутствуют. Добавляем целевое значение в копию XLSX.
     * Метрика подбирается по реальному metric.file_name.
     */
    private fun xlsxWithMetric(value: Double): MetricFixture {
        val metric = jdbcTemplate.queryForList(
            """
                SELECT id, file_name FROM metric
                WHERE file_name IS NOT NULL AND trim(file_name) <> ''
                ORDER BY id LIMIT 1
            """.trimIndent()
        ).firstOrNull()
            ?: error(
                "В тестовой PostgreSQL нет справочника metric с file_name. " +
                    "Этот справочник должен быть загружен тестовым Liquibase. " +
                    "Добавь справочную метрику в тестовые данные."
            )

        val metricId = (metric.getValue("id") as Number).toLong()
        val metricFileName = metric.getValue("file_name") as String
        val (modifiedFile, agentId) = editFixture { sheet, targetRow ->
            val sectionRow = sheet.getRow(0)
                ?: error("В тестовом XLSX отсутствует строка групп")
            val columnHeader = sheet.getRow(1)
                ?: error("В тестовом XLSX отсутствует строка заголовков")
            val formatter = DataFormatter()
            val frontalColumn = (0 until sectionRow.lastCellNum.toInt()).firstOrNull { column ->
                formatter.formatCellValue(sectionRow.getCell(column)) == FRONTAL_CELL_INDEX
            } ?: error("Для теста метрик нужен маркер группы FRONTAL_CELL_INDEX")
            val metricsColumn = (0 until sectionRow.lastCellNum.toInt()).firstOrNull { column ->
                formatter.formatCellValue(sectionRow.getCell(column)) == METRICS_CELL_INDEX
            }

            val targetColumn = if (metricsColumn != null && metricsColumn < frontalColumn) {
                // Используем существующую метриковую колонку, не меняя структуру.
                metricsColumn
            } else {
                // В этой fixture группы метрик нет: создаём ОДНУ колонку перед
                // фронтальными платформами, сохранив остальные данные shiftColumns.
                val lastColumn = sheet.asSequence()
                    .maxOfOrNull { it.lastCellNum.toInt() - 1 }
                    ?: frontalColumn
                sheet.shiftColumns(frontalColumn, lastColumn, 1)
                sectionRow.getCell(frontalColumn, Row.MissingCellPolicy.CREATE_NULL_AS_BLANK)
                    .setCellValue(METRICS_CELL_INDEX)
                frontalColumn
            }

            columnHeader.getCell(targetColumn, Row.MissingCellPolicy.CREATE_NULL_AS_BLANK)
                .setCellValue(metricFileName)
            // Заполняем выбранную метрику только для нужного агента.
            // Другие строки не должны случайно получить такую же метрику.
            for (rowIndex in 2..sheet.lastRowNum) {
                val row = sheet.getRow(rowIndex) ?: continue
                row.getCell(targetColumn, Row.MissingCellPolicy.CREATE_NULL_AS_BLANK)
                    .setCellValue("")
            }
            targetRow.getCell(targetColumn, Row.MissingCellPolicy.CREATE_NULL_AS_BLANK)
                .setCellValue(value)
        }

        return MetricFixture(modifiedFile, agentId, metricId, BigDecimal.valueOf(value))
    }

    private fun metricRow(metric: MetricFixture): Map<String, Any> {
        return jdbcTemplate.queryForList(
            """
                SELECT am.id, am.current_value, am.created, am.updated
                FROM applied_metric am
                JOIN ai_agent a ON a.id = am.ai_agent_id
                WHERE a.agent_id = ? AND am.metric_id = ?
                ORDER BY am.id
            """.trimIndent(),
            metric.agentId, metric.metricId
        ).singleOrNull() ?: error(
            "Импорт не создал ожидаемую метрику: " +
                "agentId=${metric.agentId}, metricId=${metric.metricId}"
        )
    }

    private fun contactRow(agentId: String, type: String, email: String): Map<String, Any> {
        return jdbcTemplate.queryForList(
            """
                SELECT ac.id, ac.agent_id, ac.contact_id, ac.type, ac.user_id
                FROM agent_contact ac
                JOIN ai_agent a ON a.id = ac.agent_id
                JOIN contact c ON c.id = ac.contact_id
                WHERE a.agent_id = ? AND ac.type = ? AND c.email = ?
                ORDER BY ac.id
            """.trimIndent(), agentId, type, email
        ).singleOrNull() ?: error(
            "Импорт не создал нужный контакт: agentId=$agentId, type=$type, email=$email"
        )
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

    @Test
    fun `should preserve unchanged metric ids and audit timestamps on repeated import`() {
        assertEquals(FileUploadStatus.SUCCESS, waitUntilFinishedAndUnlocked(startImport()).status)

        val metricFile = xlsxWithMetric(42.0)
        val first = waitUntilFinishedAndUnlocked(startImport(metricFile.file))
        assertEquals(FileUploadStatus.SUCCESS, first.status, first.errorDescription)
        val before = metricRow(metricFile)

        val second = waitUntilFinishedAndUnlocked(startImport(metricFile.file))
        assertEquals(FileUploadStatus.SUCCESS, second.status, second.errorDescription)
        val after = metricRow(metricFile)

        assertEquals(before["id"], after["id"], "Неизменённая метрика пересоздана")
        assertEquals(before["created"], after["created"], "created у неизменённой метрики изменился")
        assertEquals(before["updated"], after["updated"], "updated у неизменённой метрики изменился")
        assertEquals(
            0,
            (after.getValue("current_value") as BigDecimal).compareTo(metricFile.expectedValue)
        )
    }

    @Test
    fun `should update changed metric value without replacing its row`() {
        assertEquals(FileUploadStatus.SUCCESS, waitUntilFinishedAndUnlocked(startImport()).status)

        val firstMetric = xlsxWithMetric(42.0)
        val first = waitUntilFinishedAndUnlocked(startImport(firstMetric.file))
        assertEquals(FileUploadStatus.SUCCESS, first.status, first.errorDescription)
        val before = metricRow(firstMetric)

        val changedMetric = xlsxWithMetric(43.0)
        assertEquals(firstMetric.agentId, changedMetric.agentId)
        assertEquals(firstMetric.metricId, changedMetric.metricId)
        val second = waitUntilFinishedAndUnlocked(startImport(changedMetric.file))
        assertEquals(FileUploadStatus.SUCCESS, second.status, second.errorDescription)
        val after = metricRow(changedMetric)

        assertEquals(before["id"], after["id"], "Изменившаяся метрика получила новый ID")
        assertEquals(before["created"], after["created"], "Изменившаяся метрика пересоздалась")
        assertEquals(0, (after.getValue("current_value") as BigDecimal).compareTo(BigDecimal("43.0")))
    }

    @Test
    fun `should preserve imported contact row but clear legacy non PULT user ownership`() {
        assertEquals(FileUploadStatus.SUCCESS, waitUntilFinishedAndUnlocked(startImport()).status)

        val (file, agentId) = xlsxWithContacts()
        val first = waitUntilFinishedAndUnlocked(startImport(file))
        assertEquals(FileUploadStatus.SUCCESS, first.status, first.errorDescription)
        val before = contactRow(agentId, "customer", customerTestContact)
        val originalId = (before.getValue("id") as Number).toLong()

        jdbcTemplate.update("UPDATE agent_contact SET user_id = ? WHERE id = ?", 987654321L, originalId)

        val second = waitUntilFinishedAndUnlocked(startImport(file))
        assertEquals(FileUploadStatus.SUCCESS, second.status, second.errorDescription)
        val after = contactRow(agentId, "customer", customerTestContact)

        assertEquals(originalId, (after.getValue("id") as Number).toLong())
        assertEquals(before["contact_id"], after["contact_id"])
        assertEquals(before["agent_id"], after["agent_id"])
        assertNull(after["user_id"], "Старая user_id должна сбрасываться без пересоздания связи")
    }

    @Test
    fun `should remove obsolete non PULT contact links but retain unchanged ones`() {
        assertEquals(FileUploadStatus.SUCCESS, waitUntilFinishedAndUnlocked(startImport()).status)

        val (twoContactsFile, agentId) = xlsxWithContacts()
        val first = waitUntilFinishedAndUnlocked(startImport(twoContactsFile))
        assertEquals(FileUploadStatus.SUCCESS, first.status, first.errorDescription)

        val customerBefore = contactRow(agentId, "customer", customerTestContact)
        val developerBefore = contactRow(agentId, "developer", developerTestContact)
        val customerId = (customerBefore.getValue("id") as Number).toLong()
        val developerId = (developerBefore.getValue("id") as Number).toLong()

        val (customerOnlyFile, sameAgentId) = xlsxWithContacts(includeDeveloper = false)
        assertEquals(agentId, sameAgentId)
        val second = waitUntilFinishedAndUnlocked(startImport(customerOnlyFile))
        assertEquals(FileUploadStatus.SUCCESS, second.status, second.errorDescription)

        val customerAfter = contactRow(agentId, "customer", customerTestContact)
        assertEquals(customerId, (customerAfter.getValue("id") as Number).toLong())

        val obsoleteLinks = jdbcTemplate.queryForObject(
            "SELECT count(*) FROM agent_contact WHERE id = ?", Int::class.java, developerId
        ) ?: -1
        assertEquals(0, obsoleteLinks, "Устаревшая связь developer не удалена")
    }

}


```
