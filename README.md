```java

@Component
class StatusSlaUpdater(
    private val agentStatusSlaRepository: AgentStatusSlaRepository,
    private val statusRepository: StatusRepository,
    private val messageProvider: MessageProvider,
) {

    fun update(
        agent: AIAgentEntity,
        rawStatusSla: Any?,
    ): List<StatusSlaDto> {
        val agentId = requireNotNull(agent.id) {
            "AI-agent must be persisted before statusSla update"
        }

        val requestedStatusSla = parseStatusSla(
            rawStatusSla = rawStatusSla,
        )

        validateDuplicateStatuses(
            requestedStatusSla = requestedStatusSla,
        )

        val existingStatusSla =
            agentStatusSlaRepository.findAllByAiAgentId(
                agentId = agentId,
            )

        if (requestedStatusSla.isEmpty()) {
            if (existingStatusSla.isNotEmpty()) {
                agentStatusSlaRepository.deleteAll(existingStatusSla)
            }

            return emptyList()
        }

        val requestedStatusCodes: Set<String> = requestedStatusSla
            .map { statusSla ->
                requireNotNull(statusSla.status)
            }
            .toSet()

        val activeStatusesByCode: Map<String, StatusEntity> =
            statusRepository
                .findAllByDisabledIsFalse()
                .mapNotNull { statusEntity ->
                    statusEntity.code?.let { code ->
                        code to statusEntity
                    }
                }
                .toMap()

        validateStatusesExist(
            requestedCodes = requestedStatusCodes,
            statusesByCode = activeStatusesByCode,
        )

        val existingStatusSlaByStatusId = existingStatusSla
            .associateBy { entity ->
                entity.primaryKey.agentStatusId
            }

        val requestedStatusIds = mutableSetOf<Long>()
        val entitiesToSave = mutableListOf<AgentStatusSlaEntity>()
        val delta = mutableListOf<StatusSlaDto>()

        requestedStatusSla.forEach { requestedItem ->
            val statusCode = requireNotNull(requestedItem.status)
            val requestedPlannedDate =
                requireNotNull(requestedItem.plannedDate)

            val statusEntity = requireNotNull(
                activeStatusesByCode[statusCode]
            )

            val statusId = requireNotNull(statusEntity.id)

            requestedStatusIds.add(statusId)

            val existingEntity =
                existingStatusSlaByStatusId[statusId]

            if (existingEntity == null) {
                val newEntity = AgentStatusSlaEntity().apply {
                    aiAgent = agent
                    agentStatus = statusEntity
                    plannedDate = requestedPlannedDate
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
                existingEntity.plannedDate != requestedPlannedDate

            if (plannedDateChanged) {
                existingEntity.plannedDate = requestedPlannedDate

                entitiesToSave.add(existingEntity)

                delta.add(
                    StatusSlaDto(
                        status = statusCode,
                        plannedDate = requestedPlannedDate,
                    )
                )
            }
        }

        val entitiesToDelete = existingStatusSla.filter { existingEntity ->
            existingEntity.primaryKey.agentStatusId !in requestedStatusIds
        }

        if (entitiesToSave.isNotEmpty()) {
            agentStatusSlaRepository.saveAll(entitiesToSave)
        }

        if (entitiesToDelete.isNotEmpty()) {
            agentStatusSlaRepository.deleteAll(entitiesToDelete)
        }

        return delta
    }

    private fun parseStatusSla(
        rawStatusSla: Any?,
    ): List<StatusSlaDto> {
        if (rawStatusSla == null) {
            return emptyList()
        }

        val statusSlaList = rawStatusSla as? List<*>
            ?: throwWrongStatusSlaValue()

        return statusSlaList.map { rawItem ->
            val item = rawItem as? Map<*, *>
                ?: throwWrongStatusSlaValue()

            val status = item[STATUS_FIELD] as? String

            if (status.isNullOrBlank()) {
                throwWrongStatusSlaValue()
            }

            val plannedDate = parseLocalDate(
                value = item[PLANNED_DATE_FIELD],
            )

            StatusSlaDto(
                status = status,
                plannedDate = plannedDate,
            )
        }
    }

    private fun parseLocalDate(
        value: Any?,
    ): LocalDate {
        return when (value) {
            is LocalDate -> value

            is String -> {
                if (value.isBlank()) {
                    throwWrongStatusSlaValue()
                }

                try {
                    LocalDate.parse(value)
                } catch (exception: Exception) {
                    throwWrongStatusSlaValue()
                }
            }

            else -> throwWrongStatusSlaValue()
        }
    }

    private fun validateDuplicateStatuses(
        requestedStatusSla: List<StatusSlaDto>,
    ) {
        val duplicateStatuses = requestedStatusSla
            .map { statusSla ->
                requireNotNull(statusSla.status)
            }
            .groupingBy { statusCode ->
                statusCode
            }
            .eachCount()
            .filterValues { count ->
                count > 1
            }
            .keys

        if (duplicateStatuses.isNotEmpty()) {
            throw AiBadRequestException(
                errorCode = DUPLICATE_STATUS_SLA,
                message = MessageFormat.format(
                    messageProvider[DUPLICATE_STATUS_SLA],
                    duplicateStatuses.joinToString(),
                ),
            )
        }
    }

    private fun validateStatusesExist(
        requestedCodes: Set<String>,
        statusesByCode: Map<String, StatusEntity>,
    ) {
        val missingStatusCodes = requestedCodes.filterNot { code ->
            statusesByCode.containsKey(code)
        }

        if (missingStatusCodes.isNotEmpty()) {
            throw AiBadRequestException(
                errorCode = REFERENCE_VALUE_NOT_FOUND,
                message = MessageFormat.format(
                    messageProvider[REFERENCE_VALUE_NOT_FOUND],
                    missingStatusCodes.joinToString(),
                ),
            )
        }
    }

    private fun throwWrongStatusSlaValue(): Nothing {
        throw AiBadRequestException(
            errorCode = WRONG_STATUS_SLA_VALUE,
            message = messageProvider[WRONG_STATUS_SLA_VALUE],
        )
    }

    private companion object {
        const val STATUS_FIELD = "status"
        const val PLANNED_DATE_FIELD = "plannedDate"

        const val WRONG_STATUS_SLA_VALUE =
            "wrong.status.sla.value"

        const val DUPLICATE_STATUS_SLA =
            "duplicate.status.sla"

        const val REFERENCE_VALUE_NOT_FOUND =
            "reference.value.not.found"
    }
}
```
