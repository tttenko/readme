```java

requestedStatusSla.forEach { requestedItem ->
    val statusCode = requireNotNull(requestedItem.status)
    val requestedPlannedDate = requireNotNull(
        requestedItem.plannedDate
    )

    val statusEntity = requireNotNull(
        activeStatusesByCode[statusCode]
    )

    val statusId = requireNotNull(statusEntity.id)

    requestedStatusIds.add(statusId)

    val existingEntity =
        existingStatusSlaByStatusId[statusId]

    val requestedPlannedDateTime =
        requestedPlannedDate.atStartOfDay()

    if (existingEntity == null) {
        val newEntity = AgentStatusSlaEntity().apply {
            aiAgent = agent
            agentStatus = statusEntity
            plannedDate = requestedPlannedDateTime
        }

        entitiesToSave.add(newEntity)

        delta.add(
            StatusSlaDto(
                status = statusCode,
                plannedDate = requestedPlannedDate,
            )
        )

        return@forEach
    }

    val plannedDateChanged =
        existingEntity.plannedDate?.toLocalDate() !=
            requestedPlannedDate

    if (plannedDateChanged) {
        existingEntity.plannedDate =
            requestedPlannedDateTime

        entitiesToSave.add(existingEntity)

        delta.add(
            StatusSlaDto(
                status = statusCode,
                plannedDate = requestedPlannedDate,
            )
        )
    }
}
```
