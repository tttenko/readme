```java
select conname
from pg_constraint
where conrelid = 'initiative_metric_value'::regclass
  and contype = 'u';
```
