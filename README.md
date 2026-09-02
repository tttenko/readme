```java
Изучи AGENTS.md и skills/change-management.

Примени approved change proposal docs/changes/initiative-export-portfolio/proposal.md согласно локальному change-management workflow.

Перед изменениями покажи план apply.

Во время apply:

назначь placeholders реальные ID на основании актуального состояния целевых документов;
примени изменения в product.md, requirements.md, ui.md, api.md, technical.md;
обнови все cross-reference и Traceability;
для technical.md следуй tasks.md;
перед фиксацией технического решения изучи существующий admin-export в services/prm-ai-backend, если сервис подключён;
не придумывай классы, properties или архитектуру, которых нет в коде;
существующий GET /api/v1/admin/ai-agent-download не менять по контракту и поведению;
заполни Assigned IDs в proposal;
выполни documentation-review и validation;
после успешного apply выполни действия по архивированию proposal согласно change-management.

Код backend не изменять — сейчас выполняется только apply документации
```
