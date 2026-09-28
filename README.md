```java

/**
 * Проверяет наличие активной PENDING-заявки для assignment.
 *
 * Скрытые заявки с isVisibleInOffice=false считаются завершёнными
 * для текущего процесса и не препятствуют созданию новой заявки.
 */
fun existsByInitiativeMetricAssignmentIdAndStatusAndIsVisibleInOfficeTrue(
    initiativeMetricAssignmentId: Long,
    status: MetricApplicabilityRequestStatus,
): Boolean
```
