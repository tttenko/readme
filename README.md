```java
В документации для таблицы `initiative_meta_ai_object` поле `crossgoal` указано как `VARCHAR(255)` с внешним ключом на `ai_agent`.

Предлагаю использовать существующий подход проекта: хранить `initiative_id BIGINT` с FK на `ai_agent.id`, аналогично таблице `initiative_ci_solution`.

Можем ли мы изменить структуру таким образом? Есть ли бизнес-причины хранить именно строковый `crossgoal`?
```
