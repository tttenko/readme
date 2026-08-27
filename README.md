```java

delete from division
where code = 'integration-division';

delete from block
where code = 'integration-block';

delete from quality_gate
where code in (
    'QG_ARCHITECTURE',
    'QG_SECURITY',
    'DEVELOPMENT_STAGE'
);
```
