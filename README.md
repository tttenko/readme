```java

@Query(
    value = """
        SELECT DISTINCT r.assignment_id
        FROM metric_applicability_request r
        WHERE r.assignment_id IN (:assignmentIds)
          AND r.status IN ('PENDING', 'APPROVED')
          AND r.effective_from_period <= :periodMonth
          AND (r.effective_to_period IS NULL OR :periodMonth < r.effective_to_period)
    """,
    nativeQuery = true
)
fun findUnlinkedAssignmentIdsForPeriod(
    @Param("assignmentIds") assignmentIds: List<Long>,
    @Param("periodMonth") periodMonth: LocalDate,
): List<Long>
```
