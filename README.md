```java
select
    id,
    change_type,
    payload,
    created
from jira_change
where agent_id = 545
order by id desc
limit 2;
```
