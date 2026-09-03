```java
Проведи финальную доработку реализации API-014 GET /api/v1/ai-agent/initiatives/export по результатам code review.

Текущая архитектура и бизнес-логика утверждены. Не перепроектируй решение и не меняй существующий admin-export. Нужно внести только перечисленные ниже изменения, обновить тесты и затем запустить проверки.

Основные требования остаются:

FR-027 — выгружаются все инициативы, включая disabled;
FR-028 — ровно 28 колонок: первые 17 полностью совместимы с существующей admin-выгрузкой + 11 новых;
FR-029 — статусы этапов;
FR-030 — дедлайны из plannedDate;
BR-006 — endpoint доступен только TRANSFORMATION_OFFICE;
API-014 — XLSX media type и точное имя Портфель AI-инициатив.xlsx.

Не изменять:

AdminDownloadController;
AgentsReportService;
AIAgentExcelExportModel;
AIAgentExcelMapper;
ExcelExportHelper;
существующий /api/v1/admin/ai-agent-download;
JPA entities без реальной необходимости;
существующую семантику первых 17 колонок.

Выполни следующие изменения:

Persistence context для export

На InitiativeRegistryExportService.downloadRegistryExcel() добавь:

@Transactional(readOnly = true)

Причина: при формировании первых 17 колонок существующий toAIAgentExcelExportModel() читает lazy-связи jiraIssues, platforms и другие association, а новые поля используют agentContact.

Не пытайся в рамках этой правки полностью переработать загрузку AIAgentEntity.

Bulk-загрузка AgentStatusSlaEntity через:

agentStatusSlaRepository.findAllByAiAgentIdIn(initiativeIds)

должна остаться.

Не возвращайся к entity.agentStatusSla для получения SLA.

Потенциальный N+1 остальных существующих association не решай большим рефакторингом без необходимости: существующий admin mapper использует те же relations. Главное сейчас — обеспечить корректный persistence context. Если при тестировании обнаружится фактическая проблема производительности, отдельно сообщи её.

Усилить валидацию URL template

Сейчас проверяется только isBlank(). Этого недостаточно.

prm.pult.initiative-url-template должен:

быть непустым;
содержать placeholder {id}.

Например:

private fun validateInitiativeUrlTemplate(urlTemplate: String) {
    if (urlTemplate.isBlank() || !urlTemplate.contains("{id}")) {
        throw AiInternalServerException(
            message = "Property 'prm.pult.initiative-url-template' must contain '{id}'"
        )
    }
}

Вызвать в начале downloadRegistryExcel().

Сервис не должен молча создавать одинаковый URL для всех инициатив, если {id} отсутствует.

Добавить тесты:

blank template → AiInternalServerException;
template без {id} → AiInternalServerException;
корректный template → normal export.

Сам PultProperties оставить с пустым default в Kotlin:

var initiativeUrlTemplate: String = ""

Production URL должен приходить из конфигурации/environment.

Проверь реальный application.yaml, чтобы там был обычный Spring placeholder, а не Markdown-разметка из чата.

Email — пропускать null и blank

Сейчас:

entity.agentContact
    .mapNotNull { it.contact?.email }
    .joinToString(";")

Измени так, чтобы пустые/blank email не создавали ;;:

contactEmails = entity.agentContact
    .mapNotNull { it.contact?.email?.takeIf { email -> email.isNotBlank() } }
    .joinToString(";")

Правила сохраняются:

разделитель строго ;;
пробел после ; не добавлять;
порядок не регламентируется;
дедупликацию НЕ выполнять.

Не добавляй фильтрацию по AgentContactEntity.type, потому что отдельное бизнес-правило фильтрации по type не задано.

Добавить тест, например:

["biz@mail.ru", "", "   ", "it@mail.ru"]

должно дать:

biz@mail.ru;it@mail.ru
Сделать headerColumns() private

headerColumns() — внутренняя деталь InitiativeRegistryExportService, поэтому:

private fun headerColumns(): List<ExcelColumnDescription<InitiativeRegistryExcelExportModel>>

Тесты не должны обращаться к headerColumns() напрямую. Проверяй результат через сформированный XLSX.

Усилить тест структуры XLSX

Текущий тест проверяет только:

headers[0]
headers[16]
headers.subList(17, 28)

Этого недостаточно для требования «первые 17 без изменений».

Сформируй полный expected list из всех 28 заголовков и сравни:

assertEquals(expectedHeaders, headers)

Expected headers строго:

ID AI-агента
Блок
Трайб
ТБ
Наименование
Описание
Проблема, которую решает
Текущий статус
Ссылка JIRA
CROSSGOAL
GIGAUSAGE
Тип инициативы
Аудитория
Программа AI-трансформации
Фин. эффект, руб.
Фин. эффект, ПШЕ
Поверхности
is_disable
Ссылка на инициативу в Пульт
Email контакта
Статус этапа — Концепция
Дедлайн этапа — Концепция
Статус этапа — PoC
Дедлайн этапа — PoC
Статус этапа — MVP
Дедлайн этапа — MVP
Статус этапа — Целевое решение
Дедлайн этапа — Целевое решение

Также оставить проверку:

assertEquals(28, headerRow.physicalNumberOfCells)
Точное тестирование имени файла

Текущие проверки вида:

contains("AI-")

слишком слабые.

Нужно проверить фактическое имя файла:

val disposition = ContentDisposition.parse(
    response.headers.getFirst(HttpHeaders.CONTENT_DISPOSITION)!!
)

assertEquals(
    InitiativeRegistryExportService.FILE_NAME,
    disposition.filename
)

То есть реально должно проверяться:

Портфель AI-инициатив.xlsx

Аналогично усили controller test.

Не привязывай тест к внутреннему RFC5987 encoding вида %D0...: проверяй распарсенный filename.

Точная проверка формата deadline

Формат:

2026-09-03T15:30:00.000+0300

подтверждён как корректный.

Поэтому тест:

assertTrue(result.stageDeadlinePoc!!.startsWith(...))

заменить на точный:

assertEquals(
    "2026-09-01T10:30:00.000+0300",
    result.stageDeadlinePoc
)

formatMoscowDateTime() и его существующий formatter не менять.

При отсутствующем plannedDate по-прежнему:

assertNull(...)

а Excel helper оставляет соответствующую ячейку пустой.

Security tests усилить проверкой отсутствия вызова service

Для:

PROJECT_OFFICE → 403
CMS_ADMIN → 403

после MockMvc assertion добавить:

verify(exactly = 0) {
    registryExportService.downloadRegistryExcel()
}

Для:

TRANSFORMATION_OFFICE → 200

проверить:

verify(exactly = 1) {
    registryExportService.downloadRegistryExcel()
}

Это должно подтвердить, что @PreAuthorize блокирует вызов до входа в бизнес-сервис.

Bulk SLA оставить и сохранить тест

Обязательно сохранить проверку:

verify(exactly = 1) {
    agentStatusSlaRepository.findAllByAiAgentIdIn(listOf(1L, 2L))
}

verify(exactly = 0) {
    agentStatusSlaRepository.findAllByAiAgentId(any())
}

AIAgentEntity.id имеет тип non-null Long, поэтому дополнительный mapNotNull для ID не требуется:

val initiativeIds = agents.map { it.id }

допустим.

FR-029 не менять

Подтверждено, что SLA создаются для:

analysis
development
pilot
targetSolution

Поэтому текущий lookup:

slaByStatusCode["analysis"]
slaByStatusCode["development"]
slaByStatusCode["pilot"]
slaByStatusCode["targetSolution"]

корректен.

Существующую нормализацию оставить:

research → analysis
release → targetSolution

Правило оставить:

completedDate != null → Завершён
иначе stageOrdering < currentOrdering → Завершён
иначе stageOrdering == currentOrdering → В работе
иначе → План

Уже существующие тесты research, release, completedDate override и отсутствие SLA сохранить.

Hyperlink implementation оставить

Существующий post-processing подход корректен:

private const val INITIATIVE_URL_COLUMN_INDEX = 18

Header находится в row 0, data начинается с row 1, поэтому:

for (rowIndex in 1..dataSize)

корректен.

ExcelExportHelper ради hyperlink не изменять.

Сохранить тест реального POI hyperlink:

assertNotNull(cell.hyperlink)
assertEquals(expectedUrl, cell.hyperlink.address)
assertEquals(expectedUrl, cell.stringCellValue)
Первые 17 колонок / admin export оставить без изменений

Уже подтверждено, что новый headerColumns() воспроизводит существующий AgentsReportService.headerColumns() и новый mapper использует:

entity.toAIAgentExcelExportModel(
    terBanks,
    deltaPrefix,
    sigmaPrefix
)

Не делай дополнительный рефакторинг существующего AgentsReportService ради удаления дублирования заголовков. В рамках этой задачи важнее отсутствие регрессии admin-export.

После изменений запусти тесты

Сначала targeted:

mvn test -Dtest=InitiativeRegistryMapperTest,InitiativeRegistryExportServiceTest,AIAgentInitiativeControllerTest,AIAgentInitiativeControllerSecurityTest

Затем regression:

mvn test -Dtest=AgentsReportServiceTest,AIAgentExcelMapperTest,MetricsReportServiceTest

Если Maven/Surefire не принимает список классов в таком формате в этом проекте — запусти их подходящими отдельными командами, не меняя тестовую конфигурацию проекта только ради команды.

После выполнения покажи:

какие файлы изменены;
краткий diff по каждому изменению;
результаты targeted tests;
результаты regression tests;
были ли обнаружены дополнительные проблемы.

Не исправляй сторонние проблемы проекта, не относящиеся к API-014. Если тесты падают из-за уже существующей ошибки вне scope, отдельно укажи это и приложи конкретную причину.

После этих изменений не делай дополнительного архитектурного рефакторинга без необходимости.
```
