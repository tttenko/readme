```java
1. Найдём инициативу и её режимы
SELECT
    imt.id,
    imt.ai_agent_id,
    imt.agent_type
FROM initiative_metric_type imt
ORDER BY imt.ai_agent_id, imt.agent_type;

Желательно выбрать инициативу, у которой есть:

autonomous
copilot

Если записей много:

SELECT
    imt.ai_agent_id,
    array_agg(imt.agent_type ORDER BY imt.agent_type) AS agent_types
FROM initiative_metric_type imt
WHERE imt.agent_type IN ('autonomous', 'copilot')
GROUP BY imt.ai_agent_id
ORDER BY imt.ai_agent_id;
2. Посмотрим доступные метрики
SELECT
    md.id,
    md.name,
    md.unit,
    md.direction,
    md.frequency,
    md.is_active,
    md.autonomous_applicability,
    md.copilot_applicability
FROM metrics_directory md
ORDER BY md.name;

Нужно найти четыре метрики, перечисленные в конфигурации:

pre-analytics:
  metric-names:

Пришли также сами четыре значения из application.yml или application-local.yml, потому что ручка ищет метрики строго по name.

3. Посмотрим существующие значения

После выбора initiativeId подставь его вместо 123:

SELECT
    imv.id,
    imt.ai_agent_id,
    imt.id AS initiative_metric_type_id,
    imt.agent_type,
    imv.metric_directory_id,
    md.name AS metric_name,
    md.direction,
    imv.period_month,
    imv.metric_value,
    imv.target_value
FROM initiative_metric_value imv
JOIN initiative_metric_type imt
    ON imt.id = imv.initiative_agent_type_id
JOIN metrics_directory md
    ON md.id = imv.metric_directory_id
WHERE imt.ai_agent_id = 123
ORDER BY
    imt.agent_type,
    md.name,
    imv.period_month;
4. Проверим структуру таблиц

Это необходимо из-за наследования от BasicLongEntity и BasicUUIDEntity: на изображениях не видны все физические колонки.

SELECT
    c.table_name,
    c.ordinal_position,
    c.column_name,
    c.data_type,
    c.is_nullable,
    c.column_default
FROM information_schema.columns c
WHERE c.table_schema = 'public'
  AND c.table_name IN (
      'initiative_metric_type',
      'initiative_metric_value',
      'metrics_directory'
  )
ORDER BY c.table_name, c.ordinal_position;
5. Проверим ограничения и уникальные индексы
SELECT
    tc.table_name,
    tc.constraint_name,
    tc.constraint_type,
    kcu.column_name
FROM information_schema.table_constraints tc
LEFT JOIN information_schema.key_column_usage kcu
    ON kcu.constraint_schema = tc.constraint_schema
   AND kcu.constraint_name = tc.constraint_name
WHERE tc.table_schema = 'public'
  AND tc.table_name IN (
      'initiative_metric_type',
      'initiative_metric_value',
      'metrics_directory'
  )
ORDER BY
    tc.table_name,
    tc.constraint_name,
    kcu.ordinal_position;
```
