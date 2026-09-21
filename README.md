```java
select
    metric_value,
    target_value
from initiative_metric_value
where initiative_agent_type_id = 2
  and metric_directory_id = '10000000-0000-0000-0000-000000000001'
  and period_month = date '2026-08-01';

```
