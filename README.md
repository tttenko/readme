```java
@Query(
    """
    select
        a.id as id,
        a.agentId as agentId
    from AIAgentEntity a
    where a.agentId is not null
      and a.agentId not like '%PULT%'
      and (a.importStatus is null or a.importStatus <> 'blocked')
    """
)
fun findAllNonPultAgentRefs(): List<AgentImportRefProjection>
```
