```java
Изучи актуальную SDD-спецификацию initiative-catalog после apply change initiative-export-portfolio.

Основные требования реализации:
FR-027, FR-028, FR-029, FR-030, BR-006, NFR-004, API-014.

Особенно изучи requirements.md, api.md и technical.md.

Затем полностью проанализируй существующую реализацию GET /api/v1/admin/ai-agent-download: controller → service → получение данных → DTO/model → формирование XLSX.

Также найди:

источник statusSla, plannedDate, completedDate;
текущий витринный status инициативы и его порядок;
модель контактов;
поле disabled;
существующую конфигурацию URL Пульта, если она есть.

Сопоставь текущий код с FR-027..FR-030 и API-014.

Пока код не изменяй. Предложи implementation plan с конкретными существующими классами и методами. Отдельно укажи, что можно переиспользовать из admin-export без изменения его поведения.
```
