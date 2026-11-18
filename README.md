```java

interface InitiativeMetaAiObjectRepository : JpaRepository<InitiativeMetaAiObjectEntity, Long> {

    @Query("""
        SELECT obj.ciSolution
        FROM InitiativeMetaAiObjectEntity obj
        WHERE obj.initiative.id = :initiativeId
          AND obj.isActive = true
          AND obj.ciSolution IS NOT NULL
    """)
    fun findActiveCiSolutions(@Param("initiativeId") initiativeId: Long): List<String>
}

```
