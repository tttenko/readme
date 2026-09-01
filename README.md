```java
-- Проверяем исходное состояние
select id, code, label, name, disabled
from block
where lower(code) = 'cib';


-- Временно добавляем label для теста resolver
update block
set label = 'КИБ'
where lower(code) = 'cib'
  and label is null;


-- Проверяем результат
select id, code, label, name, disabled
from block
where lower(code) = 'cib';
```
