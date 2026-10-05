```java

/*
    Очистка данных интеграционных тестов FR1 и FR2.
*/

truncate table ai_agent CASCADE;
truncate table involved_resource;
truncate table implemented_platform;

-- Сначала подразделения, затем блоки.
delete from division
where code in (
    'integration-division',
    'integration-fr2-division'
);

delete from block
where code in (
    'integration-block',
    'integration-fr2-block'
);

delete from quality_gate
where code in (
    'QG_ARCHITECTURE',
    'QG_SECURITY',
    'DEVELOPMENT_STAGE',
    'QG_ARCHITECTURE_FR2',
    'QG_SECURITY_FR2',
    'STAGE_ANALYSIS_FR2',
    'STAGE_DEVELOPMENT_FR2'
);

```
