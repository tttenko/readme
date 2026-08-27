```java

select
    code,
    name,
    type,
    regexp,
    status_code,
    ordering,
    disabled
from quality_gate
where disabled is not true
order by type, ordering;

select new_depth, update_depth, max_results
from options;
```
