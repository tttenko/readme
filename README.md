```java
select
    initiative_agent_type_id,
    metric_directory_id,
    count(*)
from initiative_metric_value
group by
    initiative_agent_type_id,
    metric_directory_id
having count(*) > 1;
```
