```java
План в целом принимаю, но перед реализацией внеси следующие изменения:

Для API-014 строго соблюдай api.md: новый endpoint должен возвращать application/vnd.openxmlformats-officedocument.spreadsheetml.sheet. Не используй application/octet-stream только ради совместимости с admin-export. Существующий admin endpoint не менять.
Имя файла строго Портфель AI-инициатив.xlsx, без timestamp.
Перед использованием AIAgentRepository.findAll() проверь, как существующий AgentsReportService получает и упорядочивает инициативы. FR-027 требует сохранить порядок существующей admin-выгрузки. Используй тот же порядок; не полагайся на неопределённый порядок findAll().
Жизненный статус каждого из четырёх этапов должен рассчитываться даже при отсутствии соответствующей записи statusSla. statusSla нужен для completedDate и plannedDate, но если SLA отсутствует, статус определяется сравнением ordering этапа с текущим витринным статусом согласно FR-029. Отсутствующий plannedDate → пустая deadline-ячейка.
EmailProperties.emailLinkProperties.linkToPortalShort используй только после проверки фактического значения и назначения property. Если это не подходящий base URL Пульта, предложи отдельную конфигурируемую property. Production URL не хардкодить.
Добавь controller/security test для API-014: TRANSFORMATION_OFFICE → 200, PROJECT_OFFICE/CMS_ADMIN → 403; отдельно проверить XLSX Content-Type и точное имя файла в Content-Disposition.
Остальную архитектуру плана сохранить. AdminDownloadController, AgentsReportService, AIAgentExcelExportModel, существующий mapper и admin endpoint по поведению не менять.

После корректировки покажи финальный implementation plan, код пока не меняй.
```
