```java
select
    id,
    code,
    label,
    block_id
from division
where label is not null
  and 'КИБ/Corporate_Lending' ilike '%' || label || '%'
order by length(label) desc;

select
    id,
    code,
    label
from block
where label is not null
  and disabled is not true
  and 'КИБ/Corporate_Lending' ilike '%' || label || '%'
order by length(label) desc;
```
