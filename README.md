```java
select
    a.id,
    a.agent_id,
    a.agent_name,
    a.block_id as agent_block_id,
    a.division_id,
    d.short_name as division_name,
    d.block_id as division_block_id,
    b.short_name as block_name
from prm_ai.ai_agent a
left join prm_ai.division d
    on d.id = a.division_id
left join prm_ai.block b
    on b.id = d.block_id
where a.id in (1473, 1474, 1475, 1476);
  ```
