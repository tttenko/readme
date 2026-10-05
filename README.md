```java

select code, name, regexp, type
from quality_gate
where disabled is not true
order by type, code;

```
