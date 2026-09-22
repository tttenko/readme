```java

/* Новая проверка неприменимости метрики */
      AND NOT EXISTS (
          SELECT 1
          FROM initiative_metric_assignment assignment
          WHERE assignment.initiative_agent_type_id = metric_type.id
            AND assignment.metric_directory_id = metric.id
            AND assignment.applicability_status <> 'ACTIVE'
      )
```
