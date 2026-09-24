```java

select table_name, ordinal_position, column_name, data_type
from information_schema.columns
where table_schema = current_schema()
  and table_name in (
      'initiative_metric_value',
      'initiative_metric_type',
      'metrics_directory',
      'initiative_metric_assignment',
      'metric_applicability_request'
  )
order by table_name, ordinal_position;
```
