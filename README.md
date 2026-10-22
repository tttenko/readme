```java
# Задание для GigaCode: подготовить статические XLSX-фикстуры и исправить ExcelImportIntegrationTest

## Цель

Переработай `ExcelImportIntegrationTest`, чтобы он **не создавал и не изменял Excel-файлы во время выполнения тестов**. Вместо этого самостоятельно подготовь реальные `.xlsx` один раз, сохрани их в `src/test/resources`, а тесты должны только читать эти файлы и вызывать настоящий `ExcelUploadService`.

**Менять бизнес-код импорта нельзя.** Не создавать новых таблиц, Liquibase changeSet, шедулеров или дополнительных сервисов. Все существующие проверки атомарности, блокировки и регрессии должны сохраниться.

## 1. Изучи существующие исходники и фикстуру

Перед созданием файлов открой в проекте:

- `ExcelImportIntegrationTest.kt` (включая `xlsxWithChangedAgentName`, `xlsxWithMetric`, `xlsxWithContacts`, `editFixture`, `findImportedAgentRow`, `metricRow`, `contactRow`);
- исходную рабочую фикстуру `src/test/resources/integration/xlsx/agents.xlsx`;
- `AgentSheetService` / парсер `PreparedExcelWorkbook`, все константы названий листов, секций и заголовков (`AGENTS_SHEET_NAME`, `AGENT_ID`, `AGENT_NAME`, `CUSTOMER_CONTACT`, `DEVELOPER_CONTACT`, `METRICS_CELL_INDEX`, `FRONTAL_CELL_INDEX`, `DATA_START_ROW_INDEX`);
- схему и тестовые данные таблиц `metric`, `ai_agent`, `contact`, `agent_contact`; конфигурацию `AbstractIntegrationTest` и `clearTables.sql`.

**Не угадывай заголовки, диапазоны колонок и названия листов.** У исходного XLSX две служебные строки: группировка и заголовки (проверить по реальному файлу). Не удаляй листы `DICT`, `PROCESSES`, лист агентов, маркеры групп, формулы и другие данные, требуемые импортом. Не редактируй `agents.xlsx`: он должен остаться рабочей базовой фикстурой.

## 2. Создай файлы в существующей структуре

Используй **уже существующую** папку `src/test/resources/integration/xlsx/`, не создавай рядом вторую `integration/excel/`.

```text
src/test/resources/integration/xlsx/
├── agents.xlsx                         # существующий, не изменять
├── agents-name-changed.xlsx            # имя одного агента изменено
├── agents-metric-42.xlsx               # метрика одного агента = 42
├── agents-metric-43.xlsx               # та же метрика того же агента = 43
├── agents-contacts-both.xlsx           # customer и developer
├── agents-contacts-customer-only.xlsx  # тот же customer, developer отсутствует
└── README.md                           # описание ожидаемых значений/ID
```

**Создай настоящие бинарные файлы XLSX**, не пустые файлы и не переименованный CSV. Можно однократно использовать скрипт на Python/openpyxl или Apache POI во время разработки; генератор **не должен вызываться при запуске интеграционных тестов**. Исходный рабочий `agents.xlsx` используй как шаблон, чтобы сохранить структуру. Все варианты должны сами по себе успешно импортироваться через текущий API. Не добавляй в фикстуры реальные персональные данные.

### Точное содержимое вариантов

1. **`agents-name-changed.xlsx`** — всё как в `agents.xlsx`, кроме `AGENT_NAME` одного заранее выбранного активного агента. Укажи в `README.md` его `AGENT_ID`, исходное и новое имя. В тесте не вычисляй новое имя на основе текущей БД.
2. **`agents-metric-42.xlsx` / `agents-metric-43.xlsx`** — одинаковая структура, **одинаковый `AGENT_ID` и одинаковая справочная метрика**; отличается **только значение** 42 → 43. Заголовок метрики должен совпадать с реально существующим `metric.file_name` в тестовой БД и находиться в реальной группе `METRICS_CELL_INDEX` перед следующим разделом. Остальные значения и агентские строки одинаковы.
3. **`agents-contacts-both.xlsx`** — у одного активного **non-PULT, non-blocked** агента заполнены допустимые колонки `CUSTOMER_CONTACT` и `DEVELOPER_CONTACT` в формате, который реально понимает `createContact` (ориентир: `Integration Customer,excel-diff-test-customer@sber.ru` и `Integration Developer,excel-diff-test-developer@sber.ru`). Сверь регистр `type` с текущим парсером. Не создавай дубли по email у других строк.
4. **`agents-contacts-customer-only.xlsx`** — точная копия предыдущего файла с тем же агентом и `customer`, но `DEVELOPER_CONTACT` у него пустой. Больше **никаких отличий**. По этим файлам проверяется сохранение ID customer-связи и удаление developer-связи.

Если базовый файл не содержит подходящего non-PULT агента, в тестовых вариантах используй один и тот же специально созданный валидный non-PULT `AGENT_ID` (например `excel-diff-test-agent`) и проверь, что его остальные значения корректны и все справочные связи разрешаются. Не подменяй PULT-правила. Укажи выбранный ID в `README.md`.

Если в тестовой БД нет подходящей строки `metric` с `file_name`, не выбирай случайную и не создавай фиктивную миграцию: подготовь требуемую справочную запись через **тестовый setup/seed** по существующей схеме, только для тестов, либо явно зафиксируй блокер. Тесты должны использовать **известную конкретную метрику**, а не `ORDER BY id LIMIT 1`, результат которого меняется от состояния БД.

## 3. Перепиши весь `ExcelImportIntegrationTest` на чтение ресурсов

Создай единый helper, например:

```kotlin
private fun getExcelFile(fileName: String): MockMultipartFile {
    val resourcePath = "/integration/xlsx/$fileName"
    val bytes = requireNotNull(javaClass.getResourceAsStream(resourcePath)) {
        "Test fixture not found: $resourcePath"
    }.use { it.readBytes() }

    return MockMultipartFile(
        "file", fileName,
        "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
        bytes
    )
}
```

Используй classpath, а не `Paths.get("src/test/resources/...")`, чтобы тесты не зависели от рабочей директории Gradle/IDE.

Перепривяжи сценарии:

| Тест | Файлы |
| --- | --- |
| `should import same Excel twice without changing business data or increasing counts` | `agents.xlsx` дважды |
| `should update agent name from modified Excel and keep metrics unchanged` | `agents.xlsx` → `agents-name-changed.xlsx` |
| `should preserve unchanged metric ids and audit timestamps on repeated import` | `agents-metric-42.xlsx` дважды |
| `should update changed metric value without replacing its row` | `agents-metric-42.xlsx` → `agents-metric-43.xlsx` |
| `should preserve imported contact row but clear legacy non PULT user ownership` | `agents-contacts-both.xlsx` дважды; между импортами установить `user_id` у найденной связи |
| `should remove obsolete non PULT contact links but retain unchanged ones` | `agents-contacts-both.xlsx` → `agents-contacts-customer-only.xlsx` |

Остальные тесты (rollback, ошибки подготовки, две конкурентные регистрации, фехтование блокировки) **сохрани и адаптируй только загрузку XLSX там, где это нужно**. Не удаляй их и не ослабляй `assert`.

Удалить runtime-генераторы/модификаторы (`xlsxWithMetric`, `xlsxWithContacts`, `xlsxWithChangedAgentName`, `editFixture`, `findOrCreateContactHeader`, `findImportedAgentRow` и другие, если больше не нужны), а также неиспользуемые `XSSFWorkbook`, `DataFormatter`, `ByteArrayOutputStream` и импорты Apache POI. Оставь только методы проверки `metricRow`, `contactRow` и прочие реально необходимые SQL/assert helpers. Для ожидаемых значений используй явные константы, соответствующие `README.md`.

## 4. Тестовые данные и изоляция

- Каждый тест должен работать с чистым, воспроизводимым состоянием БД и не зависеть от порядка запуска. Не подменяй отсутствующие контакт/метрику проверкой пустой таблицы.
- После каждого импорта дождись не только `SUCCESS`/`ERROR`, но и освобождения глобальной блокировки, как уже реализовано в тесте.
- В проверке контактов `customer` должен сохранить `agent_contact.id`; `user_id` после повторного импорта non-PULT должен стать `NULL`; отсутствующий `developer` должен быть удалён.
- В проверке метрики при повторе значения 42 сохраняются `id`, `created`, `updated`; при переходе 42 → 43 значение обновляется на том же ID.
- Поскольку старый код/оптимизированный код по-разному управляет техническими датами, проверяй только согласованное поведение: не ослабляй проверки ID неизменённых строк.
- Не добавляй в production Liquibase/таблицы/шедулеры. Если нужно заполнить справочники, используй только тестовый seed с полным управлением cleanup.

## 5. Приёмка результата

1. Все шесть `.xlsx` из таблицы существуют в `src/test/resources/integration/xlsx/`, открываются как XLSX, содержат необходимые листы и дают ожидаемые значения.
2. `README.md` описывает конкретные `AGENT_ID`, имя, `metric.file_name` / `metric.id`, обе контактные email-строки и различия между файлами.
3. В `ExcelImportIntegrationTest` **нет генерации или редактирования Excel** и нет зависимости от случайной первой строки таблицы `metric`.
4. **Все существующие интеграционные тесты класса проходят**, включая четыре ранее падавших. При ошибке теста исправь фикстуру или тестовую подготовку — **не меняй бизнес-логику импорта** ради тестов.
5. Приложи список созданных файлов, изменённых Kotlin-файлов и итог запуска тестов (прошло/ошибки). Проверку выполнять с настоящим встроенным PostgreSQL из `AbstractIntegrationTest`.

**Требуемый результат:** самостоятельно создать и сохранить бинарные XLSX в репозитории, переписать интеграционный тест на эти ресурсы, выполнить тесты и прислать финальный diff и отчёт. Не ограничивайся описанием шагов: изменения должны быть внесены непосредственно в проект.


```
