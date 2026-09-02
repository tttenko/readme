```java
План принимаю. Перед началом реализации внеси две финальные технические корректировки и после этого можешь реализовывать без повторного согласования плана.

Для statusSla используй bulk-загрузку через существующий AgentStatusSlaRepository.findAllByAiAgentIdIn(...). Добавь repository в зависимости InitiativeRegistryExportService, одним запросом получи SLA для всех экспортируемых инициатив и сгруппируй их по initiative ID/status code. Не полагайся на lazy relation AIAgentEntity.agentStatusSla и не создавай N+1 запросов.
Перед реализацией hyperlink проверь фактический API ExcelExportHelper.writeSheetData и ExcelColumnDescription. Если существующий helper не предоставляет доступ к POI Cell, общий ExcelExportHelper НЕ изменяй. Сначала сформируй XLSX через существующий helper, затем в InitiativeRegistryExportService отдельным post-processing проходом установи hyperlink на ячейки колонки «Ссылка на инициативу в Пульт» через workbook.creationHelper.createHyperlink(...) / cell.hyperlink.

Добавь тест, который подтверждает bulk-вызов findAllByAiAgentIdIn и отсутствие необходимости получать SLA отдельно для каждой инициативы.

Если проект использует environment variables для URL, prm.pult.initiative-url-template вынеси в env-переменную согласно существующему стилю конфигурации проекта; не придумывай новый стиль, сначала посмотри соседние properties.

findAll() для инициатив оставить как есть — отдельный deterministic ORDER BY не нужен.

После этих изменений реализуй утверждённый план и тесты. Existing admin-export и его supporting classes не менять.
```
