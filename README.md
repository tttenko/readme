```java
select
    column_name,
    column_default,
    is_identity
from information_schema.columns
where table_name = 'enabler'
  and column_name = 'id';

```
