```java
select
    d.id as division_id,
    d.code as division_code,
    d.label as division_label,
    b.id as block_id,
    b.code as block_code,
    b.label as block_label
from division d
left join block b on b.id = d.block_id
where lower(d.label) = lower('РГС');

```
