```java
Нужно уточнить маппинг финансовых эффектов, которые шедулер будет создавать в `initiative_financial_effect` вместо заполнения полей в `ai_agent`.

Правильно ли использовать такие правила:

- `customfield_30401` → `type_code = INCOME_FINANCIAL_EFFECT`;
- `customfield_34300` → `type_code = STAFF_REDUCTIONS_FINANCIAL_EFFECT`;
- значение Jira-поля → `plan_value`;
- `fact_value = NULL`;
- `effect_year` = год создания инициативы;
- `target_month` = `31.12` соответствующего `effect_year`;
- `approval_links = []`;
- `disabled = false`;
- если соответствующее Jira-поле пустое, карточку финансового эффекта не создаём;
- `created_by` — ID технического пользователя, обозначающего создание шедулером;
- `updated_by = NULL` при первоначальном создании.

И отдельно: если шедулер впоследствии обновляет уже существующий финансовый эффект, нужно ли проставлять в `updated_by` того же технического пользователя?


```
