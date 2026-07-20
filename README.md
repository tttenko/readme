```java
SELECT
    md.id AS metric_id,
    md.name AS metric_name,
    COUNT(imv.id) AS values_count
FROM metrics_directory md
LEFT JOIN initiative_metric_value imv
    ON imv.metric_directory_id = md.id
WHERE md.name IN (
    'Точность',
    'CSI',
    'Охват',
    'Скорость'
)
GROUP BY
    md.id,
    md.name
ORDER BY md.name;

Результат будет примерно таким:

metric_id	metric_name	values_count
UUID	CSI	5
UUID	Охват	0
UUID	Скорость	2
UUID	Точность	0
values_count = 0 — связей в initiative_metric_value нет;
values_count > 0 — метрика используется, просто удалить её нельзя.

Чтобы посмотреть сами связанные записи:

SELECT
    md.id AS metric_id,
    md.name AS metric_name,
    imv.*
FROM metrics_directory md
JOIN initiative_metric_value imv
    ON imv.metric_directory_id = md.id
WHERE md.name IN (
    'Точность',
    'CSI',
    'Охват',
    'Скорость'
)
ORDER BY
    md.name,
    imv.period_month;

Чтобы проверить, какие таблицы вообще имеют внешние ключи на metrics_directory:

SELECT
    tc.constraint_name,
    tc.table_schema,
    tc.table_name AS referencing_table,
    kcu.column_name AS referencing_column,
    ccu.table_name AS referenced_table,
    ccu.column_name AS referenced_column
FROM information_schema.table_constraints tc
JOIN information_schema.key_column_usage kcu
    ON tc.constraint_name = kcu.constraint_name
    AND tc.constraint_schema = kcu.constraint_schema
JOIN information_schema.constraint_column_usage ccu
    ON tc.constraint_name = ccu.constraint_name
    AND tc.constraint_schema = ccu.constraint_schema
WHERE tc.constraint_type = 'FOREIGN KEY'
  AND ccu.table_name = 'metrics_directory';

Ожидается как минимум:

initiative_metric_value.metric_directory_id
    → metrics_directory.id

Если первый запрос покажет ненулевые связи, использовать нужно миграцию с перевязкой initiative_metric_value на новые UUID. Если все значения равны нулю и других ссылающихся таблиц нет, старые метрики можно безопасно удалить и создать заново.
```
