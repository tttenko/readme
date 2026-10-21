```java

ТЗ + полный комплект кода: упрощённая оптимизация Excel-импорта AI-агентов
Проект: prm-ai-backend; язык: Kotlin; стек: Spring Boot, Spring Data JPA, PostgreSQL, Apache POI/StreamingReader.
Цель: ускорить повторные последовательные импорты, снизить нагрузку Hibernate и сохранить атомарность всех бизнес-изменений.
Жёсткие ограничения: никаких новых таблиц, Liquibase changeSet, recovery-шедулеров, lease/fencing-generation и durable dispatcher.
0. Исполнитель — GigaCode
В этом документе ниже приведён код предлагаемых полных версий изменяемых классов. Соответствующие .kt лежат также отдельными файлами в ZIP. Не генерируй заново собственную архитектуру. Проверь существующие package/imports, актуальные константы/исключения, версии Spring и StreamingReader, затем внедряй по порядку. Не изменяй постороннюю бизнес-логику. Сначала запусти компиляцию и тесты в реальном проекте. Код составлен по присланным фрагментам, без доступа к сборке и всем внешним зависимостям проекта, поэтому не считать его скомпилированной в проекте реализацией до проверки.

Граница эквивалентности: конечные значения бизнес-полей, статусная логика, связи, blocked/PULT, текущая семантика agent.updated, SLA, QualityGate, деактивации. При сравнении/повторном импорте оптимизируем физические операции с БД. Сохранение прежних первичных ключей и created/updated у неизменившихся метрик и контактов является сознательным эффектом diff; до внедрения проверить отсутствие потребителей, которым нужна именно прежняя смена PK/даты на каждый импорт. Если такие потребители обнаружатся — приостановить этап diff и согласовать правило. Не подменять смысл agent.updated.
1. Схема исполнения
HTTP POST Excel
  ├─ filename / MIME / размер / лимит строк (без долгой транзакции)
  ├─ XLSX во временный файл, вместо file.bytes
  └─ ExcelImportRegistrationService.register()  [КОРОТКАЯ TX]
       ├─ CAS global upload_permission.upload_allowed: true→false
       │    last_updated := микросекундный token
       └─ file_upload(PROCESSING)                   [1 COMMIT]
               │
               ▼ single-worker bounded executor
  ExcelImportPreparationService.prepare()         [БЕЗ БИЗНЕС-TX]
       ├─ Excel StreamingReader → типизированные PreparedRow
       ├─ временные файлы для трёх листов Excel (без БД и без heap на 10000 строк)
       └─ единственный внешний запрос справочника Тербанков
               │
               ▼ ExcelImportApplyService.apply() [ОДНА TX]
       ├─ pg_advisory_xact_lock (один фиксированный ключ)
       ├─ DictAndProcessService.applyPreparedDictionaries  [MANDATORY]
       ├─ EntityManager.flush()
       ├─ AgentSheetService.processAgentSheet [MANDATORY + REQUIRED Template]
       │    ├─ batch 100; fetch-platform/resources/contacts отдельными запросами
       │    ├─ метрики batch-load + diff
       │    ├─ контакты managed collections + orphanRemoval
       │    ├─ удалить контакты отсутствующих/дублей
       │    ├─ SLA/деактивация/статусы/QualityGate
       │    └─ file_upload.SUCCESS
       ├─ flush + SELECT upload_permission FOR UPDATE & compare token
       └─ COMMIT ВСЕХ бизнес-изменений или ROLLBACK ВСЕХ
               │
               ▼
       ERROR после rollback отдельно / условный release только владельцем
       Удаление временных файлов в finally
Staging: нормализованные содержимое ячеек хранится во временных файлах на файловой системе pod, НЕ в таблицах PostgreSQL. Это промежуточное хранение реальных разобранных данных, а не просто сохранение file.bytes. Полная проверка ссылочной бизнес-целостности остаётся внутри атомарного apply; ошибка приводит к rollback, внешне не появляется частичный результат. Автоматическое возобновление после смерти pod не поддерживается — пользователь загружает XLSX повторно, оператор безопасно разблокирует GLOBAL. Требование «все ошибки должны обнаруживаться именно ДО открытия транзакции» этой облегчённой схемой НЕ обеспечивается; если оно обязательно, нужен отдельный чистый семантический валидатор до apply.
2. Заменить / добавить / удалить
Добавить: PreparedExcelWorkbook.kt, ExcelImportPreparationService.kt, ExcelImportRegistrationService.kt, ExcelImportApplyService.kt, RegisteredExcelImport (в файле регистрации), PreparedExcelImport (в файле подготовки).

Заменить существующие: ExcelUploadService.kt, UploadPermissionService.kt, UploadPermissionRepository.kt, AgentSheetService.kt, DictAndProcessService.kt, AIAgentRepository.kt (внести только новые методы, остальные сохранить), AppliedMetricRepository.kt (новый batch-метод), AgentContactRepository.kt, ProcessRepository.kt (добавить IS DISTINCT FROM), FileUploadStatusService.kt (сохранить все внешние методы).

Не трогать без необходимости: Entity, составные PK, все остальные Repository, контроллеры GET/list, DTO API и сервисы Jira/PULT/FR1/FR2. Сервисный код не должен присваивать null обязательным FK.

Удалить все НЕПРИМЕНЁННЫЕ новые объекты GigaCode: AiaImportStagingEntity, staging repositories, AiaImportLifecycleService, AiaImportDurableDispatch, AiaImportWorker, AiaImportProperties, AiaImportRegisteredEvent, AiaImportPhase и все createTable/addColumn миграции из новой реализации. Удалить их include из Liquibase master. Не удалять существующие таблицы БД. Удостовериться, что не осталось ссылок на AiaImport* из конфигурации, тестов и контроллеров.
Важные требования совместимости
- ExcelUploadService не должен иметь @Transactional на upload(). ExcelImportRegistrationService.register() — отдельный Spring bean, его commit завершается до executor.execute.
- У AgentSheetService есть внутренний TransactionTemplate: проверить, что propagation REQUIRED (не REQUIRES_NEW). Дополнительная аннотация MANDATORY блокирует запуск вне apply. То же у справочников.
- В новом DictAndProcessService внешний HTTP к Тербанкам вызывается на prepare, а не внутри большой транзакции.
- Изменения словарей перед entityManager.clear() обязательно flush. Иначе dirty checking может потерять изменения.
- AIAgentRepository.fetch... выполняются отдельно для каждой inverse-коллекции: не использовать один JOIN FETCH сразу по нескольким List.
- AppliedMetricsEntity не имеет явного уникального ключа по (ai_agent_id, metric_id): не использовать associateBy без обработки дублей; текущий diff использует очереди на metric.id.
- Сохранять политику удаления метрик всех типов: старая загрузка уничтожала метрики разных типов; не сохранять пользовательские метрики без отдельного согласования.
- Сохранять глобальную очистку контактов для отсутствующих в Excel, дублей, не-PULT и не-blocked. Только у обработанных агентов реализуем managed diff; для PULT оставлено прежнее поведение.
- Не устранять автоматическое обновление agent.updated, involvedResource.updated или дат SLA без отдельной бизнес-проверки.
- Текущее @Fetch(FetchMode.JOIN) у QualityGate может добавлять JOIN и накладные расходы. Менять глобальный маппинг только после изучения ВСЕХ потребителей.
- BasicLongEntity использует GenerationType.IDENTITY, поэтому установка Hibernate jdbc.batch_size не решает проблему batch INSERT для IDENTITY-сущностей. Миграции ID-стратегии запрещены.
3. Блокировка и аварийное восстановление
При регистрации CAS UPDATE WHERE upload_allowed=true; при release сравнить токен с last_updated. Последний DB-запрос в apply — SELECT ... FOR UPDATE существующей строки GLOBAL, проверка токена. Дополнительно transaction-scoped advisory lock сериализует apply даже при ошибочной ручной разблокировке.

Поскольку отдельной колонки owner_token не будет, last_updated применяется как идентификатор текущего захвата. Это компромисс без DDL: поле имеет точность времени и применяется ещё как аудит. При наличии внешних прямых обновлений upload_permission они должны всегда изменять last_updated. Прежде чем выпускать в production, провести конкурирующие тесты (два POST, ручной unlock, старый worker ещё выполняется). Для криптографически уникального токена отдельное поле в существующей таблице было бы надёжнее, но по условию оно не добавляется.

Не использовать прежний isUploadAllowed() + disableUpload() (TOCTOU) и fail-open catch Exception → true. Нет GLOBAL/недоступна БД — ошибка, а не разрешение нового импорта.

Административный endpoint unlockFileUpload(userId) сохранён: вместо безусловной разблокировки он допускает изменение GLOBAL лишь при отсутствии записей file_upload.status=PROCESSING во всей БД. После crash оператор сначала проверяет отсутствие живых задач и переводит зависший PROCESSING в ERROR. SQL-проверка сама по себе не знает о потоке, ошибочно помеченном ERROR извне: запрещены force-unlock на живом pod.

После подтверждённого падения worker и остановки активных задач можно восстановить право так:
BEGIN;
UPDATE file_upload
SET status='error', error_description='Interrupted by worker/pod restart', uploaded=clock_timestamp()
WHERE id = :stale_file_upload_id AND status='processing';
UPDATE upload_permission
SET upload_allowed=true, last_updated=clock_timestamp(), updated_by=:operator_user_id
WHERE id='GLOBAL' AND upload_allowed=false;
COMMIT;
Не выполнять этот SQL при реально работающем apply; проверять pg_stat_activity, текущие pod, запросы и лог PROCESSING.
4. Приоритет внедрения
1. Компилируемый каркас и атомарность. Заменить ExcelUploadService, регистрацию, блокировку, preparation и apply. Проверить, что изменения словарей и агентов коммитятся одной tx. Сохранить существующие статусы API.
2. Typed staging и прежние парсеры. Подставить PreparedRow/PreparedCell/PreparedSheet в сервисы; сверить форматирование даты, типы формул и исходную семантику пропусков Excel.
3. Пакетные загрузки и метрики. Предзагрузка метрик по 100 agentId, diff вместо per-agent DELETE+INSERT, проверка дубликатов/всех типов/деактивации.
4. Контакты/связи. Fetch коллекций в том же persistence context, diff с orphanRemoval, пакетная очистка отсутствующих, сохранение PULT/blocked.
5. Словари/процессы. Map для кодов, dirty checking, отдельный QG/статус SQL только при наличии изменений в рамках исходного бизнес-контракта.
6. Интеграционные, конкурентные и нагрузочные тесты. Сравнение БД до/после и 5–6 последовательных запусков одного файла. Без этого перехода в DEV/production нет.
5. Обязательные тесты
1. Первая загрузка (словарь, агент, метрики, контакты, SLA, QG, статусы).
2. Повтор ТОГО ЖЕ Excel 5–6 раз: одинаковое состояние бизнес-данных, число метрик/связей не растёт, SUCCESS у всех 6.
3. Один изменённый параметр метрики; добавление/удаление, разные scale BigDecimal; две метрики с одинаковым fileName; метрики других типов — сравнение со старым поведением.
4. Частичный конфликт agentId и blocked/PULT; устаревший агент, присутствующий дубль, контакты агента, отсутствующего в Excel.
5. Преднамеренная ошибка в конце файла после изменения словарей: полный rollback всей бизнес-БД, ERROR отдельно.
6. Падение подготовки: в бизнес-таблицах не меняется НИЧЕГО.
7. Два параллельных запроса регистрации (CountDownLatch): только одна запись с PROCESSING и один владелец GLOBAL.
8. Сброс GLOBAL при работающем старом worker: прежний apply обязан откатиться на финальной проверке; новый не должен перекрыться по commit.
9. Штатное завершение: release условный; чужую блокировку не снять.
10. Границы 99/100/101 и 499/500/501; ошибки в любой пачке, результат атомарен.
11. Формулы, даты, пустые ячейки, отсутствие листа, перепутанные заголовки, строки с пробелами, валидные файлы до лимита 10000.
12. Замеры до/после: длительность prepare/apply/commit, count SELECT/INSERT/UPDATE/DELETE, N+1, Heap/RSS/GC, HikariPool, ожидания PostgreSQL, k8s OOMKilled.
Стоп-факторы: нельзя считать проверкой лишь HTTP 200 с id: он означает регистрацию, не SUCCESS. Нельзя помечать ERROR после успешного commit из-за сбоя удаления temp-файла. Нельзя вызывать saveAll всех метрик при неизменном файле.
6. Риски и вопросы перед merge
- При падении pod PROCESSING и глобальный lock останутся до ручного вмешательства — это сознательный выбор упрощённой архитектуры.
- Temp-файлы должны размещаться на ephemeral storage с гарантированным свободным объёмом, автоматическим удалением при нормальном завершении, правами процесса и без логирования содержимого.
- Ограничение maxCells=2_000_000 защищает от XLSX с экстремальным количеством ячеек, но может быть новым ограничением. Сделать конфигурируемым и сверить с максимальным допустимым реальным файлом, прежде чем запускать в production.
- Не выводить содержимое Excel, персональные контакты, конфигурационные секреты в логах.
- В присланной конфигурации есть открытый пароль БД: перед передачей/PR удалить и сменить.
7. Полные Kotlin-файлы
Далее идут полные тела заменяемых и добавляемых классов. Блоки не содержат package, поскольку пакет проекта в присланных фрагментах отсутствует: GigaCode обязан перенести сохранённые объявления package, существующие импорты и добавить новые. Несвязанные с импортом методы Repository сохраняются как в оригинале.
PreparedExcelWorkbook.kt
// Поместить в пакет service.upload.prepare; скорректировать импорты пакета проекта.
import com.monitorjbl.xlsx.StreamingReader // Сверить с фактическим импортом проекта
import org.apache.poi.ss.usermodel.Cell
import org.apache.poi.ss.usermodel.CellType
import org.apache.poi.ss.usermodel.DateUtil
import java.io.BufferedInputStream
import java.io.BufferedOutputStream
import java.io.Closeable
import java.io.EOFException
import java.io.ObjectInputStream
import java.io.ObjectOutputStream
import java.io.Serializable
import java.nio.file.Files
import java.nio.file.Path
import java.util.Date

/** Подготовленные типизированные ячейки. Java serialization применяется только к НАШИМ объектам. */
data class PreparedCell(
    val columnIndex: Int,
    val rowIndex: Int,
    val cellType: CellType,
    val cachedFormulaResultType: CellType,
    private val textValue: String?,
    private val numberValue: Double?,
    private val booleanValue: Boolean?,
    private val dateMillis: Long?,
    val isDateFormatted: Boolean
) : Serializable {
    val stringCellValue: String get() = textValue
        ?: throw IllegalStateException("Non-string Excel cell at row=$rowIndex, column=$columnIndex")
    val numericCellValue: Double get() = numberValue
        ?: throw IllegalStateException("Non-numeric Excel cell at row=$rowIndex, column=$columnIndex")
    val booleanCellValue: Boolean get() = booleanValue
        ?: throw IllegalStateException("Non-boolean Excel cell at row=$rowIndex, column=$columnIndex")
    val dateCellValue: Date? get() = dateMillis?.let(::Date)

    companion object {
        fun capture(cell: Cell): PreparedCell {
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
                isDateFormatted = formattedDate
            )
        }
    }
}

/** Не содержит Hibernate/JPA-сущностей; подготовленную строку можно читать многократно. */
class PreparedRow(val rowNum: Int, private val cells: List<PreparedCell>) : Iterable<PreparedCell>, Serializable {
    @Transient private var indexedCells: Map<Int, PreparedCell>? = null
    fun getCell(index: Int): PreparedCell? {
        val indexByColumn = indexedCells ?: cells.associateBy(PreparedCell::columnIndex).also { indexedCells = it }
        return indexByColumn[index]
    }
    override fun iterator(): Iterator<PreparedCell> = cells.iterator()
}

/** Бинарная временная таблица строк одного листа на диске; не является таблицей PostgreSQL. */
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

/** Хранилище подготовленных Excel-строк. Ограничено по памяти: строки на диске, в памяти лишь batch. */
class PreparedExcelWorkbook(private val sheets: Map<String, PreparedSheet>, private val paths: List<Path>) : Closeable {
    fun getSheet(name: String): PreparedSheet = sheets[name]
        ?: throw IllegalArgumentException("Missing required Excel sheet: $name")

    override fun close() {
        paths.forEach { path -> runCatching { Files.deleteIfExists(path) } }
    }

    companion object {
        fun create(source: Path, requiredSheetNames: List<String>, maxCells: Long = 2_000_000): PreparedExcelWorkbook {
            val sheetPaths = mutableListOf<Path>()
            val result = linkedMapOf<String, PreparedSheet>()
            var totalCells = 0L
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
                                        totalCells++
                                        if (totalCells > maxCells) {
                                            throw IllegalArgumentException("Too many populated Excel cells: $totalCells")
                                        }
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

ExcelImportPreparationService.kt
/** Промежуточное хранение нормализованных ячеек вне основной бизнес-транзакции. */
@Service
class ExcelImportPreparationService(
    private val internalTerBanksApiClient: InternalTerBanksApiClient
) {
    fun prepare(source: Path): PreparedExcelImport {
        val workbook = PreparedExcelWorkbook.create(
            source = source,
            requiredSheetNames = listOf(DICT, PROCESSES, AGENTS_SHEET_NAME)
        )
        try {
            // Никаких обращений к внешнему API внутри длинного apply/COMMIT.
            val terBanks = internalTerBanksApiClient.findAll().body
            return PreparedExcelImport(workbook, terBanks)
        } catch (failure: Exception) {
            workbook.close()
            throw failure
        }
    }
}

/** Содержит только независимые от JPA данные, подготовленные до изменения основных таблиц. */
class PreparedExcelImport(
    val workbook: PreparedExcelWorkbook,
    val terBanksRefs: MutableList<TerBank>?
) : AutoCloseable {
    override fun close() = workbook.close()
}

ExcelImportRegistrationService.kt
/** Регистрация ID и захват права фиксируются одним COMMIT до отправки в executor. */
@Service
class ExcelImportRegistrationService(
    private val fileUploadRepository: FileUploadRepository,
    private val uploadPermissionService: UploadPermissionService,
    private val messageProvider: MessageProvider
) {
    @Transactional
    fun register(fileName: String, userId: Long): RegisteredExcelImport {
        val acquiredAt = uploadPermissionService.acquire(userId)
            ?: throw AiResponseException(
                HttpStatus.FORBIDDEN,
                FILE_UPLOAD_NOT_AllOWED,
                message = messageProvider[FILE_UPLOAD_NOT_AllOWED]
            )
        val entity = fileUploadRepository.saveAndFlush(
            FileUploadEntity(
                fileName = fileName,
                status = PROCESSING,
                errorDescription = "Processing started...",
                uploaded = LocalDateTime.now(),
                createdBy = userId
            )
        )
        return RegisteredExcelImport(entity.id, acquiredAt, userId, fileName)
    }
}

data class RegisteredExcelImport(
    val fileUploadId: Long,
    val acquiredAt: LocalDateTime,
    val userId: Long,
    val fileName: String
)

UploadPermissionRepository.kt
// В существующем UploadPermissionRepository заменить только методы управления блокировкой.
// НИКАКИХ новых столбцов и миграций. last_updated — токен владения текущей загрузкой.
@Repository
interface UploadPermissionRepository : JpaRepository<UploadPermission, String> {
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
    fun tryAcquire(@Param("acquiredAt") acquiredAt: LocalDateTime,
                   @Param("userId") userId: Long): Int

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
    fun releaseIfOwner(@Param("acquiredAt") acquiredAt: LocalDateTime,
                       @Param("userId") userId: Long): Int

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

    @Modifying
    @Query(value = """
        UPDATE upload_permission
        SET upload_allowed = true,
            last_updated = clock_timestamp(),
            updated_by = :userId
        WHERE id = 'GLOBAL' AND upload_allowed = false
          AND NOT EXISTS (
              SELECT 1 FROM file_upload
              WHERE lower(status) = 'processing'
          )
    """, nativeQuery = true)
    fun unlockOnlyWhenNoProcessing(@Param("userId") userId: Long): Int

    @Query(value = "SELECT upload_allowed FROM upload_permission WHERE id='GLOBAL'", nativeQuery = true)
    fun readPermissionFlag(): Boolean?

}

UploadPermissionService.kt
/** Транзакционный CAS через существующую upload_permission. Без lease/планировщиков. */
@Service
class UploadPermissionService(
    private val permissionRepository: UploadPermissionRepository
) {
    /** Выполнить внутри транзакции регистрации file_upload. */
    @Transactional(propagation = Propagation.MANDATORY)
    fun acquire(userId: Long): LocalDateTime? {
        // PostgreSQL TIMESTAMP хранит микросекунды; округляем до той же точности.
        val acquiredAt = LocalDateTime.now().truncatedTo(java.time.temporal.ChronoUnit.MICROS)
        return when (permissionRepository.tryAcquire(acquiredAt, userId)) {
            1 -> acquiredAt
            0 -> {
                if (!permissionRepository.existsById("GLOBAL")) {
                    throw IllegalStateException("upload_permission.GLOBAL does not exist")
                }
                null
            }
            else -> throw IllegalStateException("More than one GLOBAL upload permission row")
        }
    }

    /** Вызвать последним DB-действием основной apply-транзакции. */
    @Transactional(propagation = Propagation.MANDATORY)
    fun confirmBeforeCommit(acquiredAt: LocalDateTime) {
        if (permissionRepository.stillOwnerForUpdate(acquiredAt) != true) {
            throw IllegalStateException("Excel import lost GLOBAL ownership before commit")
        }
    }

    /** Отдельная короткая транзакция после commit/rollback; чужой lock не освобождает. */
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    fun release(acquiredAt: LocalDateTime, userId: Long) {
        permissionRepository.releaseIfOwner(acquiredAt, userId)
    }

    /** Сохраняем административный API, но не разрешаем сбросить lock живого импорта. */
    @Transactional
    fun administrativeUnlock(userId: Long) {
        if (permissionRepository.unlockOnlyWhenNoProcessing(userId) == 1) return
        if (permissionRepository.readPermissionFlag() == true) return // уже свободно
        throw IllegalStateException(
            "Cannot unlock Excel import while PROCESSING exists; check pods and mark stale upload ERROR first"
        )
    }

}

ExcelImportApplyService.kt
/** Единственный сервис, который транзакционно меняет бизнес-таблицы. */
@Service
class ExcelImportApplyService(
    private val dictAndProcessService: DictAndProcessService,
    private val agentSheetService: AgentSheetService,
    private val uploadPermissionService: UploadPermissionService,
    private val entityManager: EntityManager
) {
    @Transactional
    fun apply(prepared: PreparedExcelImport, registration: RegisteredExcelImport) {
        // Сериализуем атомарные apply даже при ошибочной ручной разблокировке.
        // Используется один фиксированный ключ ТОЛЬКО для данного импорта.
        entityManager.createNativeQuery("SELECT pg_advisory_xact_lock(80162026)").singleResult

        // Справочники и процессы больше не имеют REQUIRES_NEW.
        val context = dictAndProcessService.applyPreparedDictionaries(
            workbook = prepared.workbook,
            fileUploadId = registration.fileUploadId,
            userId = registration.userId,
            terBanksRefs = prepared.terBanksRefs
        )
        entityManager.flush() // до AgentSheetService, который очищает persistence context

        // Внутренний TransactionTemplate ДОЛЖЕН быть PROPAGATION_REQUIRED.
        // AgentSheetService также записывает file_upload.SUCCESS и blockedAgents.
        agentSheetService.processAgentSheet(
            context = context,
            agentSheet = prepared.workbook.getSheet(AGENTS_SHEET_NAME),
            userId = registration.userId
        )

        entityManager.flush()
        uploadPermissionService.confirmBeforeCommit(registration.acquiredAt)
        // Spring COMMIT при нормальном выходе, ROLLBACK на исключение.
    }
}

ExcelUploadService.kt
/**
 * Единственный сценарий импорта без staging-таблиц и recovery-шедулеров:
 * синхронная быстрая валидация -> транзакционная регистрация -> один bounded worker
 * -> дисковая подготовка -> единый бизнес COMMIT -> условная разблокировка.
 */
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
    private val executor = java.util.concurrent.ThreadPoolExecutor(
        1, 1, 0L, java.util.concurrent.TimeUnit.MILLISECONDS,
        java.util.concurrent.SynchronousQueue(),
        java.util.concurrent.ThreadFactory { task -> Thread(task, "excel-import-worker").apply { isDaemon = true } },
        java.util.concurrent.ThreadPoolExecutor.AbortPolicy()
    )

    fun getFileUploadInfo(id: Long): FileUploadInfoResponse = FileUploadInfoResponse(
        fileUploadRepository.findByIdOrNull(id) ?: throw AiNotFoundException(
            errorCode = FILE_UPLOAD_NOT_FOUND,
            message = messageProvider[FILE_UPLOAD_NOT_FOUND]
        )
    )

    /** Здесь НЕТ @Transactional: нельзя запускать worker до commit регистрации. */
    fun upload(file: MultipartFile, userId: Long): Long {
        if (!localFileTypeValidator.isFileTypeValid(file)) {
            throw PrmResponseException(HttpStatus.BAD_REQUEST, FileNotValid, messageProvider[FileNotValid])
        }
        if (file.size > 20L * 1024 * 1024) {
            throw AiResponseException(HttpStatus.BAD_REQUEST, AI_UPLOAD_MAX_FILE_SIZE,
                MessageFormat.format(messageProvider[AI_UPLOAD_MAX_FILE_SIZE]))
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
            } catch (rejected: java.util.concurrent.RejectedExecutionException) {
                // Регистрация уже зафиксирована, поэтому ERROR — отдельной транзакцией.
                runCatching { fileUploadStatusService.updateStatus(registered.fileUploadId, ERROR,
                    "Import executor rejected task", registered.userId) }
                    .onFailure { log.error("Could not set ERROR after rejected import", it) }
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

    private fun validateAgentRowCount(path: java.nio.file.Path) {
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
                            throw AiResponseException(HttpStatus.BAD_REQUEST, AI_UPLOAD_MAX_ROWS_SIZE,
                                MessageFormat.format(messageProvider[AI_UPLOAD_MAX_ROWS_SIZE]))
                        }
                    }
                    if (DATA_START_ROW_INDEX >= rows - 1) {
                        throw AiResponseException(HttpStatus.BAD_REQUEST, AI_UPLOAD_EMPTY_FILE,
                            MessageFormat.format(messageProvider[AI_UPLOAD_EMPTY_FILE]))
                    }
                }
            }
        } catch (known: AiResponseException) {
            throw known
        } catch (failure: Exception) {
            throw AiResponseException(HttpStatus.BAD_REQUEST, AI_UPLOAD_CORRUPTED_FILE,
                MessageFormat.format(messageProvider[AI_UPLOAD_CORRUPTED_FILE]))
        }
    }

    private fun processAsync(source: java.nio.file.Path, registration: RegisteredExcelImport) {
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
            // Без System.gc(): JVM самостоятельно управляет heap.
        }
    }

    /** Контракт старого контроллера сохранён; безусловного force-unlock больше нет. */
    fun unlockFileUpload(userId: Long) {
        uploadPermissionService.administrativeUnlock(userId)
    }

    @PreDestroy
    fun shutdown() { executor.shutdown() }

    // Сохранить существующую административную ручку разблокировки, но не позволять
    // ей обходить ownership при работающем worker. Для аварийного восстановления —
    // ручной SQL только после проверки, что активной загрузки уже нет.
}


DictAndProcessService.kt
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
            val programsByCode = availablePrograms.associateBy { it.code?.trim()?.lowercase() }.toMutableMap()
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
            val processesByName = availableProcesses.associateBy { it.shortName?.trim()?.lowercase() }.toMutableMap()
            val blocksByCode = availableBlocks.associateBy { it.code }
            val programsByCode = availablePrograms.associateBy { it.code }
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
                    }).also { availableProcesses.add(it); processesByName[processFromDict.shortName.trim().lowercase()] = it }
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
        isDisabled: (ENTITY) -> Boolean,
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
            val platformsByCode = availablePlatforms.associateBy { it.code?.trim()?.lowercase() }.toMutableMap()
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

AgentSheetService.kt
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
        if (metricsStartIndex == -1 || metricsEndIndex == -1 || headerRow == null) return

        val reusableByMetricId = existingMetrics
            .filter { it.type == STANDARD && it.metric?.id != null }
            .groupBy { it.metric!!.id }
            .mapValues { (_, matches) -> java.util.ArrayDeque(matches) }
        val retainedIds = mutableSetOf<Long>()
        val newMetrics = mutableListOf<AppliedMetricsEntity>()

        row.filter { it.columnIndex in metricsStartIndex..metricsEndIndex }.forEach { metricCell ->
            val metricName = headerRow.getCell(metricCell.columnIndex)?.stringCellValue?.trim()
            if (metricName.equals("Индивидуальная метрика", ignoreCase = true)) return@forEach
            val rawValue = getValueOrNull(metricCell)?.takeIf { it.isNotBlank() } ?: return@forEach
            val metricRef = metricName?.let(::normalizeKey)?.let(dictionaries.metricsByFileName::get)
                ?: throw AiFileUploadException(
                    AI_UPLOAD_UNKNOWN_METRIC_NAME,
                    MessageFormat.format(messageProvider[AI_UPLOAD_UNKNOWN_METRIC_NAME],
                        metricCell.columnIndex + 1, row.rowNum + 1), fileId = fileId
                )
            val amount = try {
                BigDecimal.valueOf(rawValue.replace(',', '.').toDouble())
            } catch (exception: NumberFormatException) {
                throw AiFileUploadException(
                    AI_UPLOAD_UNKNOWN_METRIC_NAME,
                    MessageFormat.format(messageProvider[AI_UPLOAD_UNKNOWN_METRIC_NAME],
                        metricCell.columnIndex + 1, row.rowNum + 1), fileId = fileId
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
     * Для PULT сохраняем старое поведение: старые связи не удалялись глобальным DELETE,
     * а новые добавлялись. Для остальных агентов — точечная синхронизация коллекции,
     * включая удаление дублей, с сохранением orphanRemoval.
     */
    private fun synchronizeAgentContacts(agent: AIAgentEntity, desiredContacts: List<AgentContactEntity>) {
        if (agent.agentId?.contains("PULT") == true) {
            agent.agentContact.addAll(desiredContacts)
            return
        }
        val desiredByKey = desiredContacts.associateBy { contactLink ->
            contactLink.type to contactLink.contact?.id
        }
        val retainedKeys = mutableSetOf<Pair<String?, Long?>>()
        val iterator = agent.agentContact.iterator()
        while (iterator.hasNext()) {
            val current = iterator.next()
            val key = current.type to current.contact?.id
            if (key !in desiredByKey || !retainedKeys.add(key)) {
                iterator.remove()
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

AIAgentRepository.kt
interface AIAgentRepository : JpaRepository<AIAgentEntity, Long> {

    fun findFirstByAgentId(agentId: String): AIAgentEntity?

    @Query(
        value = """
    WITH filtered_agents AS (
        SELECT DISTINCT a.*,
        CASE WHEN a.deadline_expired = 'ontime' THEN 1
             WHEN a.deadline_expired = 'expiration' THEN 2
             WHEN a.deadline_expired = 'expired' THEN 3
             ELSE 4 END as deadline_priority
        FROM ai_agent a
        LEFT JOIN status s ON a.agent_status_id = s.id
        LEFT JOIN division d ON a.division_id = d.id
        LEFT JOIN block b1 ON d.block_id = b1.id
        LEFT JOIN block b2 ON a.block_id = b2.id
        LEFT JOIN implemented_platform ip ON a.id = ip.ai_agent_id
        LEFT JOIN platform p ON ip.platform_id = p.id
        LEFT JOIN program prgm ON a.program_id = prgm.id
        LEFT JOIN initiative_type ite ON a.agent_initiative_type = ite.code
        LEFT JOIN user_audience ua ON a.user_audience_code = ua.code
        LEFT JOIN agent_contact ac ON a.id = ac.agent_id
        LEFT JOIN jira_issue ji ON ji.agent_id = a.id 
        AND LOWER(ji.project) = 'crossgoal' WHERE a.disabled = :disabled
            AND (
                :search IS NULL OR
                LOWER(a.agent_name) LIKE LOWER(CONCAT('%', :search, '%')) OR
                LOWER(a.agent_id) LIKE LOWER(CONCAT('%', :search, '%')) OR
                LOWER(a.agent_jira_url) LIKE LOWER(CONCAT('%', :search, '%')) OR
                LOWER(ji.jira_key) LIKE LOWER(CONCAT('%', :search, '%'))
            )
          AND (:terbankIds IS NULL OR a.terbank_id IN (:terbankIds))
          AND (COALESCE(:programCodes) IS NULL OR prgm.code IN (:programCodes))
          AND (COALESCE(:agentStatusCodes) IS NULL OR s.code IN (:agentStatusCodes))
          AND (COALESCE(:blockCodes) IS NULL OR (b2.code IN (:blockCodes) OR b1.code IN (:blockCodes)))
          AND (COALESCE(:divisionCodes) IS NULL OR d.code IN (:divisionCodes))
          AND (COALESCE(:initiativeTypes) IS NULL OR ite.code IN (:initiativeTypes))
          AND (COALESCE(:userAudiences) IS NULL OR ua.code IN (:userAudiences))
          AND (COALESCE(:platformCodes) IS NULL OR p.code IN (:platformCodes))
          AND (COALESCE(:deadlineExpiredIds) IS NULL OR a.deadline_expired IN (:deadlineExpiredIds))
          AND (:userId IS NULL OR a.owner_id = :userId OR ac.user_id = :userId)
          AND (:filterByDeviation = false OR a.id IN (:deviationInitiativeIds)
)
)
SELECT * FROM filtered_agents
ORDER BY
CASE
WHEN :sortField = 'alphabet' AND :sortDirection = 'ASC' THEN LOWER(agent_name)
END ASC,
CASE
WHEN :sortField = 'alphabet' AND :sortDirection = 'DESC' THEN LOWER(agent_name)
END DESC,
CASE
WHEN :sortField = 'deadline_priority' AND :sortDirection = 'ASC' THEN deadline_priority
END ASC NULLS LAST,
CASE
WHEN :sortField = 'deadline_priority' AND :sortDirection = 'DESC' THEN deadline_priority
END DESC NULLS LAST,
CASE
WHEN :sortField = 'agent_effect_optimization' AND :sortDirection = 'ASC' THEN agent_effect_optimization
END ASC NULLS LAST,
CASE
WHEN :sortField = 'agent_effect_optimization' AND :sortDirection = 'DESC' THEN agent_effect_optimization
END DESC NULLS LAST,
CASE
WHEN :sortField = 'agent_effect_revenue' AND :sortDirection = 'ASC' THEN agent_effect_revenue
END ASC NULLS LAST,
CASE
WHEN :sortField = 'agent_effect_revenue' AND :sortDirection = 'DESC' THEN agent_effect_revenue
END DESC NULLS LAST,
LOWER(agent_name) ASC NULLS LAST,
agent_id ASC
""",
countQuery = """
SELECT COUNT(DISTINCT a.id)
FROM ai_agent a
LEFT JOIN status s ON a.agent_status_id = s.id
LEFT JOIN division d ON a.division_id = d.id
LEFT JOIN block b1 ON d.block_id = b1.id
LEFT JOIN block b2 ON a.block_id = b2.id
LEFT JOIN implemented_platform ip ON a.id = ip.ai_agent_id
LEFT JOIN platform p ON ip.platform_id = p.id
LEFT JOIN program prgm ON a.program_id = prgm.id
LEFT JOIN initiative_type ite ON a.agent_initiative_type = ite.code
LEFT JOIN user_audience ua ON a.user_audience_code = ua.code
LEFT JOIN agent_contact ac ON a.id = ac.agent_id
LEFT JOIN jira_issue ji ON ji.agent_id = a.id
AND LOWER(ji.project) = 'crossgoal' WHERE a.disabled = :disabled
AND (
:search IS NULL OR
LOWER(a.agent_name) LIKE LOWER(CONCAT('%', :search, '%')) OR
LOWER(a.agent_id) LIKE LOWER(CONCAT('%', :search, '%')) OR
LOWER(a.agent_jira_url) LIKE LOWER(CONCAT('%', :search, '%')) OR
LOWER(ji.jira_key) LIKE LOWER(CONCAT('%', :search, '%'))
)
AND (:terbankIds IS NULL OR a.terbank_id IN (:terbankIds))
AND (COALESCE(:programCodes) IS NULL OR prgm.code IN (:programCodes))
AND (COALESCE(:agentStatusCodes) IS NULL OR s.code IN (:agentStatusCodes))
AND (COALESCE(:blockCodes) IS NULL OR (b2.code IN (:blockCodes) OR b1.code IN (:blockCodes)))
AND (COALESCE(:divisionCodes) IS NULL OR d.code IN (:divisionCodes))
AND (COALESCE(:platformCodes) IS NULL OR p.code IN (:platformCodes))
AND (COALESCE(:initiativeTypes) IS NULL OR ite.code IN (:initiativeTypes))
AND (COALESCE(:userAudiences) IS NULL OR ua.code IN (:userAudiences))
AND (COALESCE(:deadlineExpiredIds) IS NULL OR a.deadline_expired IN (:deadlineExpiredIds))
AND (:userId IS NULL OR a.owner_id = :userId OR ac.user_id = :userId)
AND (:filterByDeviation = false OR a.id IN (:deviationInitiativeIds)
)
""",
nativeQuery = true
)
fun findAll(
@Param("search") search: String?,
@Param("terbankIds") terbankIds: Collection<Long>?,
@Param("programCodes") programCodes: Collection<String>?,
@Param("agentStatusCodes") agentStatusCodes: Collection<String>?,
@Param("blockCodes") blockCodes: Collection<String>?,
@Param("divisionCodes") divisionCodes: Collection<String>?,
@Param("platformCodes") platformCodes: Collection<String>?,
@Param("initiativeTypes") initiativeTypes: Collection<String>?,
@Param("userAudiences") userAudiences: Collection<String>?,
@Param("disabled") disabled: Boolean,
@Param("deadlineExpiredIds") deadlineExpiredIds: Collection<String>?,
@Param("userId") userId: Long?,
@Param("sortField") sortField: String,
@Param("sortDirection") sortDirection: String,
@Param("filterByDeviation") filterByDeviation: Boolean,
@Param("deviationInitiativeIds") deviationInitiativeIds: Collection<Long>,
pageRequest: Pageable
): Page<AIAgentEntity>

    @Query(
        value = """
        SELECT DISTINCT a.id
        FROM ai_agent a
        LEFT JOIN status s ON a.agent_status_id = s.id
        LEFT JOIN division d ON a.division_id = d.id
        LEFT JOIN block b1 ON d.block_id = b1.id
        LEFT JOIN block b2 ON a.block_id = b2.id
        LEFT JOIN implemented_platform ip ON a.id = ip.ai_agent_id
        LEFT JOIN platform p ON ip.platform_id = p.id
        LEFT JOIN program prgm ON a.program_id = prgm.id
        LEFT JOIN initiative_type ite ON a.agent_initiative_type = ite.code
        LEFT JOIN user_audience ua ON a.user_audience_code = ua.code
        LEFT JOIN agent_contact ac ON a.id = ac.agent_id
        LEFT JOIN jira_issue ji ON ji.agent_id = a.id
        AND LOWER(ji.project) = 'crossgoal' WHERE a.disabled = :disabled
            AND (
            :search IS NULL
            OR LOWER(a.agent_name) LIKE LOWER(CONCAT('%', :search, '%'))
            OR LOWER(a.agent_id) LIKE LOWER(CONCAT('%', :search, '%'))
            OR LOWER(a.agent_jira_url) LIKE LOWER(CONCAT('%', :search, '%'))
            OR LOWER(ji.jira_key) LIKE LOWER(CONCAT('%', :search, '%'))
          )
          AND (:terbankIds IS NULL OR a.terbank_id IN (:terbankIds))
          AND (COALESCE(:programCodes) IS NULL OR prgm.code IN (:programCodes))
          AND (COALESCE(:agentStatusCodes) IS NULL OR s.code IN (:agentStatusCodes))
          AND (COALESCE(:blockCodes) IS NULL OR b2.code IN (:blockCodes) OR b1.code IN (:blockCodes))
          AND (COALESCE(:divisionCodes) IS NULL OR d.code IN (:divisionCodes))
          AND (COALESCE(:platformCodes) IS NULL OR p.code IN (:platformCodes))
          AND (COALESCE(:initiativeTypes) IS NULL OR ite.code IN (:initiativeTypes))
          AND (COALESCE(:userAudiences) IS NULL OR ua.code IN (:userAudiences))
          AND (COALESCE(:deadlineExpiredIds) IS NULL OR a.deadline_expired IN (:deadlineExpiredIds))
          AND (:userId IS NULL OR a.owner_id = :userId OR ac.user_id = :userId)
    """,
        nativeQuery = true,
    )
    fun findFilteredInitiativeIds(
        @Param("search") search: String?,
        @Param("terbankIds") terbankIds: Collection<Long>?,
        @Param("programCodes") programCodes: Collection<String>?,
        @Param("agentStatusCodes") agentStatusCodes: Collection<String>?,
        @Param("blockCodes") blockCodes: Collection<String>?,
        @Param("divisionCodes") divisionCodes: Collection<String>?,
        @Param("platformCodes") platformCodes: Collection<String>?,
        @Param("initiativeTypes") initiativeTypes: Collection<String>?,
        @Param("userAudiences") userAudiences: Collection<String>?,
        @Param("disabled") disabled: Boolean,
        @Param("deadlineExpiredIds") deadlineExpiredIds: Collection<String>?,
        @Param("userId") userId: Long?
    ): Set<Long>

    @Query(
        """
        SELECT MAX(
                    CASE
                        WHEN (a.updated is null) or (a.created > a.updated) THEN a.created
                        ELSE a.updated
                END) 
        FROM AIAgentEntity a
    """
    )
    fun findMaxCreatedOrUpdatedDate(): LocalDateTime?

    fun findAllByDisabledIsFalse(): List<AIAgentEntity>

    @Query(
        """
        SELECT NEW ru.sber.prm.repository.AgentStatusCountDTO(
            a.division.id,
            a.agentStatus.id,
            COUNT(a.agentStatus.id),
            CAST(a.deadlineExpired AS string),
            COUNT(a.deadlineExpired))
        FROM 
            AIAgentEntity a
        WHERE a.disabled != TRUE 
            AND (((:divisionId IS NULL)) OR (a.division.id = (:divisionId)))
            AND (((:programId IS NULL)) OR (a.program.id = (:programId)))
        GROUP BY 
            a.division.id,
            a.agentStatus.id,
            a.deadlineExpired
    """
    )
    fun findAgentStatusCounts(
        @Param("divisionId") divisionId: Long?,
        @Param("programId") programId: Long?
    ): List<AgentStatusCountDTO>

    @EntityGraph(
        attributePaths = [
            "initiativeType",
            "division",
            "block",
            "agentStatus",
            "processes",
            "enablers",
            "involvedResource",
            "agentStrategies",
            "agentStrategies.strategy"
        ]
    )
    @Query(
        """
    SELECT a
        FROM AIAgentEntity a 
        WHERE a.agentId LIKE '%PULT%'
        AND a.jiraStatus = 'pending'
        AND a.disabled != true
        ORDER BY a.id
    """
    )
    fun findAllWithPultIdAndPending(): List<AIAgentEntity>

    @Modifying
    @Transactional
    @Query(
        """
        update AIAgentEntity a
        set a.jiraStatus = coalesce(:#{#dto.jiraStatus}, a.jiraStatus), 
            a.jiraErrorCount = coalesce(:#{#dto.jiraErrorCount}, a.jiraErrorCount), 
            a.jiraUpdated = coalesce(:#{#dto.jiraUpdated}, a.jiraUpdated) 
        where a.id = :#{#dto.id}
    """
    )
    fun updateJiraSyncStatus(@Param("dto") dto: JiraSyncStatusDto)

    @Query(
        """
        SELECT a
        FROM AIAgentEntity a
        WHERE a.agentId NOT LIKE '%PULT%'
        AND a.agentJiraUrl LIKE '%CROSSGOAL-%'
        AND a.disabled != true
        ORDER BY a.id
    """
    )
    fun findAllWithPultIdAndCrossgoalPageable(pageable: Pageable): Page<AIAgentEntity>

    fun findAllByAgentIdIn(agentIds: Collection<String>): List<AIAgentEntity>

    /** Отдельные запросы: одновременный fetch нескольких коллекций создаёт cartesian explosion. */
    @Query("SELECT DISTINCT a FROM AIAgentEntity a LEFT JOIN FETCH a.platforms WHERE a.id IN :ids")
    fun fetchPlatformsForExcelImport(@Param("ids") ids: Collection<Long>): List<AIAgentEntity>

    @Query("SELECT DISTINCT a FROM AIAgentEntity a LEFT JOIN FETCH a.involvedResource WHERE a.id IN :ids")
    fun fetchResourcesForExcelImport(@Param("ids") ids: Collection<Long>): List<AIAgentEntity>

    @Query("SELECT DISTINCT a FROM AIAgentEntity a LEFT JOIN FETCH a.agentContact WHERE a.id IN :ids")
    fun fetchContactsForExcelImport(@Param("ids") ids: Collection<Long>): List<AIAgentEntity>


    @EntityGraph(
        attributePaths = [
            "initiativeType",
            "division",
            "block",
            "agentStatus",
            "processes",
            "enablers",
            "involvedResource",
            "agentStrategies",
            "agentStrategies.strategy"]
    )
    @Query(
        """
    SELECT DISTINCT a
    FROM AIAgentEntity a
    WHERE a.jiraStatus = 'pendingUpdate'
    AND a.disabled != true
    ORDER BY a.id
    """
    )
    fun findAllPendingJiraUpdate(): List<AIAgentEntity>

    @Query(
        """
    SELECT a
    FROM AIAgentEntity a
    WHERE a.jiraStatus = 'inProgress'
    ORDER BY a.id
    """
    )
    fun findAllWithJiraStatusInProgress(): List<AIAgentEntity>

    @Query(
        value = """
        SELECT *
        FROM ai_agent
        WHERE id = :id
        FOR UPDATE
    """, nativeQuery = true
    )
    fun findByIdForUpdate(@Param("id") id: Long): AIAgentEntity?

    @Query(
        """
        select
            agent.id as initiativeId,
            status.code as statusCode,
            status.ordering as statusOrdering
        from AIAgentEntity agent
        left join agent.agentStatus status
        where agent.id in :initiativeIds
    """
    )
    fun findInitiativeStatuses(@Param("initiativeIds") initiativeIds: Collection<Long>): List<InitiativeStatusProjection>

    @Query(
        """
    select
        a.id as id,
        a.agentId as agentId
    from AIAgentEntity a
    where a.agentId is not null
      and a.agentId not like '%PULT%'
      and (a.importStatus is null or a.importStatus <> 'blocked')
    """
    )
    fun findAllImportManagedNonPultAgentRefs(): List<AgentImportRefProjection>

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

AppliedMetricRepository.kt
interface AppliedMetricRepository: JpaRepository<AppliedMetricsEntity, Long> {

    @Modifying
    @Query("DELETE FROM AppliedMetricsEntity a WHERE a.aiAgent.id = :aiAgentId")
    fun deleteAllByAiAgentId(@Param("aiAgentId") aiAgentId: Long)

    /**
     * Все применённые метрики агента — для diff-сверки при импорте (Этап 2):
     * вместо DELETE-all + INSERT применяем точечный diff, чтобы итоговое
     * состояние совпадало, но без лишних удалений/вставок.
     */
    fun findAllByAiAgentId(aiAgentId: Long): List<AppliedMetricsEntity>

    @EntityGraph(attributePaths = ["metric"])
    fun findAllByAiAgentIdIn(aiAgentIds: Collection<Long>): List<AppliedMetricsEntity>
}

-- Очистка staging-таблиц импорта и сброс глобального права к дефолту (fail-closed: upload_allowed=true).
truncate table aia_import_contact_staging;
truncate table aia_import_metric_staging;
truncate table aia_import_relation_staging;
truncate table aia_import_agent_staging;
truncate table aia_import_dict_staging;
truncate table aia_import_staging;
truncate table file_upload CASCADE;

-- Восстанавливаем глобальное право: свободно, без владельца, без lease.
UPDATE upload_permission
SET upload_allowed = true,
owner_token = NULL,
lease_expires_at = NULL,
fencing_generation = COALESCE(fencing_generation, 0),
last_updated = NOW(),
updated_by = NULL
WHERE id = 'GLOBAL';



-- Имитация простоя без heartbeat: истекает lease всех подготовленных/прочих staging-импортов.
UPDATE aia_import_staging
SET lease_expires_at = NOW() - interval '1 minute',
updated_at = NOW();

AgentContactRepository.kt
@Repository
interface AgentContactRepository : JpaRepository<AgentContactEntity, Long> {
    /** Оставлено для сторонних текущих потребителей; импорт больше не вызывает. */
    @Modifying
    @Query(value = """
        DELETE FROM agent_contact ac USING ai_agent a
        WHERE ac.agent_id = a.id
          AND a.agent_id NOT LIKE '%PULT%'
          AND a.import_status IS DISTINCT FROM 'blocked'
    """, nativeQuery = true)
    fun deleteAllForImportedNonBlockedAgents(): Int

    /** Удаляются только связи агентов, которых импорт НЕ обработал. */
    @Modifying
    @Query(value = "DELETE FROM agent_contact WHERE agent_id IN (:ids)", nativeQuery = true)
    fun deleteForAgentIds(@Param("ids") agentIds: Collection<Long>): Int
}

ProcessRepository.kt
interface ProcessRepository: JpaRepository<ProcessEntity, Long> {

    fun findAllByDisabledIsFalse():List<ProcessEntity>

    @Query(
        """
            SELECT 
                DISTINCT p.*
            FROM 
                process p
            JOIN agent_process ap ON p.id = ap.process_id
            JOIN ai_agent ag ON ag.id = ap.agent_id
            JOIN block blck ON p.block_id = blck.id
            JOIN program prgm ON p.program_id = prgm.id
            JOIN status stts ON p.status_id = stts.id
            WHERE 
                (:includeDisabled = true OR p.disabled != true)
                AND ag.disabled != true 
                AND (COALESCE(:blockCodes, NULL) IS NULL OR blck.code IN (:blockCodes))
                AND (COALESCE(:programCodes, NULL) IS NULL OR prgm.code IN (:programCodes))
                AND (COALESCE(:statusCodes, NULL) IS NULL OR stts.code IN (:statusCodes))
                AND (COALESCE(:searchLike, NULL) IS NULL OR p.short_name ILIKE :searchLike OR p.name ILIKE :searchLike)
            ORDER BY p.short_name
            """,
                countQuery = """
            SELECT 
                COUNT(DISTINCT p.id)
            FROM 
                process p
            JOIN agent_process ap ON p.id = ap.process_id
            JOIN ai_agent ag ON ag.id = ap.agent_id
            JOIN block blck ON p.block_id = blck.id
            JOIN program prgm ON p.program_id = prgm.id
            JOIN status stts ON p.status_id = stts.id
            WHERE 
                (:includeDisabled = true OR p.disabled != true)
                AND ag.disabled != true 
                AND (COALESCE(:blockCodes, NULL) IS NULL OR blck.code IN (:blockCodes))
                AND (COALESCE(:programCodes, NULL) IS NULL OR prgm.code IN (:programCodes))
                AND (COALESCE(:statusCodes, NULL) IS NULL OR stts.code IN (:statusCodes))
                AND (COALESCE(:searchLike, NULL) IS NULL OR p.short_name ILIKE :searchLike OR p.name ILIKE :searchLike)
    """,
        nativeQuery = true
    )
    fun findByFilter(
        @Param("searchLike") search: String?,
        @Param("blockCodes") blockCodes: List<String>?,
        @Param("programCodes") programCodes: List<String>?,
        @Param("statusCodes") statusCodes: List<String>?,
        @Param("includeDisabled") includeDisabled: Boolean = false,
        pageable: Pageable
    ): Page<ProcessEntity>

    @Query("""
            select
                b.id as block_id,
                as2.code as status_code,
                count(p_info.id) as process_count
            from
                block b
            cross join status as2
            left join (
                select
                    distinct p.*
                from
                    process p
                join 
                agent_process ap on
                    p.id = ap.process_id
                join 
                ai_agent ag on
                    ag.id = ap.agent_id
                join "program" p2 on
                    p2.id = p.program_id
                where
                    ag.disabled != true
                    and p.disabled != true
                    and (coalesce(:programCodes, null) is null
                        or p2.code in (:programCodes))
                ) p_info on
                    p_info.status_id = as2.id
                    and p_info.block_id = b.id
            group by
                b.id,
                as2.id
            order by
                b.id,
                as2.id 
        """, nativeQuery = true)
    fun getStatusMetrics(
        @Param("by") by: String?,
        @Param("programCodes") programCodes: List<String>?
    ): List<StatusMetricView>

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
              and p.status_id is distinct from r.status_id
        """,
        nativeQuery = true
    )
    fun refreshStatusesFromAgents(): Int
}

FileUploadStatusService.kt
@Service
class FileUploadStatusService(
    private val fileUploadRepository: FileUploadRepository,
    private val userInfoProvider: UserInfoProvider
) {
    @Transactional(readOnly = true)
    fun list(): List<FileUploadInfoResponse> = fileUploadRepository.findAll()
        .map { FileUploadInfoResponse(it) }
        .sortedByDescending { it.created }
        .take(20)

    @Transactional(readOnly = true)
    fun getById(id: Long): FileUploadInfoResponse? = fileUploadRepository.findByIdOrNull(id)
        ?.let { FileUploadInfoResponse(it) }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    fun updateStatus(fileUploadId: Long, status: String, errorDescription: String? = null, updatedBy: Long? = null) {
        val entity = fileUploadRepository.findByIdOrNull(fileUploadId)
            ?: throw AiNotFoundException(message = FILE_UPLOAD_NOT_FOUND, errorCode = FILE_UPLOAD_NOT_FOUND)
        entity.status = status
        entity.errorDescription = errorDescription
        entity.updatedBy = updatedBy
        entity.uploaded = LocalDateTime.now()
        // JPA dirty checking: save(entity) не требуется.
    }


    // Публичный контракт существующего сервиса сохраняется.
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    fun delete(fileUploadId: Long) {
        fileUploadRepository.deleteById(fileUploadId)
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    fun init(file: MultipartFile, userId: Long? = null): FileUploadEntity =
        fileUploadRepository.saveAndFlush(
            FileUploadEntity(
                fileName = file.originalFilename.orEmpty(),
                status = PROCESSING,
                createdBy = userId ?: userInfoProvider.currentUser().id
            )
        )
}

ExcelImportIntegrationTest.kt
/**
 * Интеграционный регрессионный тест; использовать существующий реальный XLSX fixture.
 * Это тест без MockkBean на AgentSheetService / DictAndProcessService / ApplyService.
 * Проверить исходный fixture: он должен быть валиден по всем бизнес-правилам.
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

    @BeforeEach
    fun prepareTest() {
        mockCurrentDate(LocalDate.of(2025, 1, 1))
        every { internalTerBanksApiClient.findAll() } returns ResponseEntity.ok(
            listOf(TerBank().apply {
                id = 12
                code = 0
                name = "Центральный аппарат"
                shortName = "ЦА"
            })
        )
        jdbcTemplate.update("UPDATE upload_permission SET upload_allowed = true, last_updated = clock_timestamp() WHERE id = 'GLOBAL'")
    }

    @AfterEach
    fun cleanup() {
        // Удалять только после завершения worker; никаких TRUNCATE во время импорта.
        executeSqlScript("integration/sql/clearTables.sql")
        jdbcTemplate.update("UPDATE upload_permission SET upload_allowed = true, last_updated = clock_timestamp() WHERE id = 'GLOBAL'")
    }

    private fun xlsx(): MockMultipartFile {
        val content = Files.readAllBytes(Paths.get("src/test/resources/integration/xlsx/agents.xlsx"))
        return MockMultipartFile(
            "file", "agents.xlsx",
            "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet", content
        )
    }

    private fun waitUntilFinished(fileUploadId: Long): FileUploadEntity {
        await().atMost(120, TimeUnit.SECONDS).untilAsserted {
            val upload = fileUploadRepository.findByIdOrNull(fileUploadId)
            assertNotNull(upload)
            assertTrue(upload!!.status == SUCCESS || upload.status == ERROR)
        }
        return fileUploadRepository.findById(fileUploadId).orElseThrow()
    }

    @Test
    fun `should finish two sequential identical imports without increasing metric count`() {
        val firstId = excelUploadService.upload(xlsx(), 1L)
        val first = waitUntilFinished(firstId)
        assertEquals(SUCCESS, first.status, "First import failed: ${first.errorDescription}")
        val agentCountAfterFirst = aiAgentRepository.count()
        val metricCountAfterFirst = appliedMetricRepository.count()

        val secondId = excelUploadService.upload(xlsx(), 1L)
        val second = waitUntilFinished(secondId)
        assertEquals(SUCCESS, second.status, "Second import failed: ${second.errorDescription}")
        assertEquals(agentCountAfterFirst, aiAgentRepository.count())
        assertEquals(metricCountAfterFirst, appliedMetricRepository.count())
    }

    @Test
    fun `should reject second registration while first owns global lock`() {
        val first = registrationService.register("first.xlsx", 1L)
        try {
            assertThrows<AiResponseException> {
                registrationService.register("second.xlsx", 2L)
            }
        } finally {
            permissionService.release(first.acquiredAt, first.userId)
        }
    }
}

```
