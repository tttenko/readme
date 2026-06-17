```java
select id, author, filename, dateexecuted, orderexecuted, exectype
from databasechangelog
where lower(id) like '%autoincrement%'
   or lower(id) like '%enabler%'
order by dateexecuted desc;

```
