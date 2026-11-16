```java
-- 1. Проверяем типы и nullable
SELECT
    column_name,
    data_type,
    character_maximum_length,
    is_nullable
FROM information_schema.columns
WHERE table_name = 'ai_agent'
  AND column_name IN ('id', 'agent_id');

-- 2. Проверяем ограничения таблицы
SELECT
    con.conname AS constraint_name,
    con.contype AS constraint_type,
    pg_get_constraintdef(con.oid) AS definition
FROM pg_constraint con
JOIN pg_class rel ON rel.oid = con.conrelid
WHERE rel.relname = 'ai_agent'
  AND con.contype IN ('p', 'u', 'f')
ORDER BY con.contype, con.conname;

-- 3. Проверяем индексы
SELECT
    indexname,
    indexdef
FROM pg_indexes
WHERE tablename = 'ai_agent';
```
