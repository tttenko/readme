```java

select
    id,
    author,
    filename,
    dateexecuted,
    orderexecuted,
    exectype
from databasechangelog
where id = 'Update pending metric applicability request unique index'
  and author = 'KoptenkoMV';
```
