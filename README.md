```java

@Component
open class StatusSlaUpdater(
    private val agentStatusSlaRepository: AgentStatusSlaRepository,
    private val statusRepository: StatusRepository,
    private val messageProvider: MessageProvider,
) {

    open fun update(
        agent: AIAgentEntity,
        rawStatusSlaValue: Any?,
    ): List<StatusSlaDto> {

        val persistedAgentId = requireNotNull(agent.id) {
            "AI-agent must be persisted before statusSla update"
        }

        val requestedStatusSlaItems = parseStatusSla(
            rawStatusSla = rawStatusSlaValue,
        )

        val persistedStatusSlaEntities =
            agentStatusSlaRepository.findAllByAiAgentId(
                agentId = persistedAgentId,
            )

        if (requestedStatusSlaItems.isEmpty()) {
            if (persistedStatusSlaEntities.isNotEmpty()) {
                agentStatusSlaRepository.deleteAll(
                    entities = persistedStatusSlaEntities,
                )
            }

            return emptyList()
        }

        val requestedStatusCodes = requestedStatusSlaItems
            .map { requestedStatusSla ->
                requireNotNull(requestedStatusSla.status)
            }
            .toSet()

        val activeStatusesByCode = statusRepository
            .findAllByDisabledIsFalse()
            .mapNotNull { activeStatus ->
                activeStatus.code?.let { statusCode ->
                    statusCode to activeStatus
                }
            }
            .toMap()

        validateRequestedStatusesExist(
            requestedStatusCodes = requestedStatusCodes,
            activeStatusesByCode = activeStatusesByCode,
        )

        val persistedStatusSlaByStatusId =
            persistedStatusSlaEntities.associateBy { persistedStatusSla ->
                persistedStatusSla.primaryKey.agentStatusId
            }

        val requestedStatusIds = mutableSetOf<Long>()

        val statusSlaEntitiesToSave =
            mutableListOf<AgentStatusSlaEntity>()

        val changedStatusSlaItems =
            mutableListOf<StatusSlaDto>()

        requestedStatusSlaItems.forEach { requestedStatusSla ->

            val requestedStatusCode =
                requireNotNull(requestedStatusSla.status)

            val requestedPlannedDate =
                requireNotNull(requestedStatusSla.plannedDate)

            val requestedStatusEntity = requireNotNull(
                activeStatusesByCode[requestedStatusCode]
            )

            val requestedStatusId =
                requireNotNull(requestedStatusEntity.id)

            requestedStatusIds.add(requestedStatusId)

            val persistedStatusSla =
                persistedStatusSlaByStatusId[requestedStatusId]

            val requestedPlannedDateTime =
                requestedPlannedDate.atStartOfDay()

            if (persistedStatusSla == null) {
                val newStatusSlaEntity =
                    AgentStatusSlaEntity().apply {
                        aiAgent = agent
                        agentStatus = requestedStatusEntity
                        plannedDate = requestedPlannedDateTime
                    }

                statusSlaEntitiesToSave.add(newStatusSlaEntity)

                changedStatusSlaItems.add(
                    StatusSlaDto(
                        status = requestedStatusCode,
                        plannedDate = requestedPlannedDate,
                    )
                )

                return@forEach
            }

            val hasPlannedDateChanged =
                persistedStatusSla.plannedDate?.toLocalDate() !=
                    requestedPlannedDate

            if (hasPlannedDateChanged) {
                persistedStatusSla.plannedDate =
                    requestedPlannedDateTime

                statusSlaEntitiesToSave.add(persistedStatusSla)

                changedStatusSlaItems.add(
                    StatusSlaDto(
                        status = requestedStatusCode,
                        plannedDate = requestedPlannedDate,
                    )
                )
            }
        }

        val statusSlaEntitiesToDelete =
            persistedStatusSlaEntities.filter { persistedStatusSla ->
                persistedStatusSla.primaryKey.agentStatusId !in
                    requestedStatusIds
            }

        if (statusSlaEntitiesToSave.isNotEmpty()) {
            agentStatusSlaRepository.saveAll(
                entities = statusSlaEntitiesToSave,
            )
        }

        if (statusSlaEntitiesToDelete.isNotEmpty()) {
            agentStatusSlaRepository.deleteAll(
                entities = statusSlaEntitiesToDelete,
            )
        }

        return changedStatusSlaItems
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
                } catch (_: DateTimeParseException) {
                    throwWrongStatusSlaValue()
                }
            }

            else -> throwWrongStatusSlaValue()
        }
    }

    private fun validateRequestedStatusesExist(
        requestedStatusCodes: Set<String>,
        activeStatusesByCode: Map<String, StatusEntity>,
    ) {
        val missingStatusCodes =
            requestedStatusCodes.filterNot { requestedStatusCode ->
                activeStatusesByCode.containsKey(requestedStatusCode)
            }

        if (missingStatusCodes.isNotEmpty()) {
            throw AiBadRequestException(
                errorCode = PROCESS_WITH_ID_NOT_FOUND,
                message = MessageFormat.format(
                    messageProvider[PROCESS_WITH_ID_NOT_FOUND],
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
    }
} 
```
