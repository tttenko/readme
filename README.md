```java

select id, code, label
from division
where label is not null
  and (
      lower(label) like '%corporate%'
      or lower(label) like '%lending%'
      or lower(label) like '%киб%'
  );

select id, code, label
from block
where label is not null
  and (
      lower(label) like '%corporate%'
      or lower(label) like '%lending%'
      or lower(label) like '%киб%'
  );
```
