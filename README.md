```java

@Component
class StatusSlaUpdater(
    private val agentStatusSlaRepository: AgentStatusSlaRepository,
    private val statusRepository: StatusRepository,
    private val messageProvider: MessageProvider,
) {

    /**
     * Синхронизирует записи agent_status_sla с блоком statusSla из PATCH-запроса.
     *
     * Возвращает только дельту:
     * 1. новые сроки;
     * 2. существующие сроки, у которых изменилась календарная дата.
     *
     * completedDate существующих записей не изменяется.
     */
    fun update(
        agent: AIAgentEntity,
        rawStatusSla: Any?,
    ): List<StatusSlaDto> {
        val agentId = requireNotNull(agent.id) {
            "AI-agent must be persisted before statusSla update"
        }

        /*
         * Сначала полностью парсим и валидируем запрос.
         * До этого момента состояние БД не изменяется.
         */
        val requestedStatusSla = parseStatusSla(
            rawStatusSla = rawStatusSla,
        )

        validateDuplicateStatuses(
            requestedStatusSla = requestedStatusSla,
        )

        /*
         * Одним запросом получаем текущие SLA агента.
         */
        val existingStatusSla =
            agentStatusSlaRepository.findAllByAiAgentId(
                agentId = agentId,
            )

        /*
         * Если statusSla = null или statusSla = [],
         * удаляем все связанные с агентом сроки.
         */
        if (requestedStatusSla.isEmpty()) {
            if (existingStatusSla.isNotEmpty()) {
                agentStatusSlaRepository.deleteAll(existingStatusSla)
            }

            return emptyList()
        }

        /*
         * Одним запросом получаем все активные статусы справочника
         * и индексируем их по code.
         */
        val activeStatusesByCode = statusRepository
            .findAllByDisabledIsFalse()
            .mapNotNull { status ->
                status.code?.let { code -> code to status }
            }
            .toMap()

        val requestedStatusCodes = requestedStatusSla
            .map { statusSla -> statusSla.status }
            .toSet()

        validateStatusesExist(
            requestedCodes = requestedStatusCodes,
            statusesByCode = activeStatusesByCode,
        )

        /*
         * Текущие SLA индексируем по agent_status_id,
         * который является частью составного первичного ключа.
         */
        val existingStatusSlaByStatusId = existingStatusSla
            .associateBy { entity ->
                entity.primaryKey.agentStatusId
            }

        val requestedStatusIds = mutableSetOf<Long>()

        /*
         * Здесь одновременно собираются:
         * - новые сущности;
         * - существующие сущности с изменённой plannedDate.
         *
         * В конце будет один вызов saveAll().
         */
        val entitiesToSave = mutableListOf<AgentStatusSlaEntity>()

        /*
         * Дельта для последующего создания jira_change
         * с changeType = plannedDate.
         */
        val delta = mutableListOf<StatusSlaDto>()

        requestedStatusSla.forEach { requestedItem ->
            val statusEntity = requireNotNull(
                activeStatusesByCode[requestedItem.status]
            )

            val statusId = requireNotNull(statusEntity.id)

            requestedStatusIds.add(statusId)

            val existingEntity =
                existingStatusSlaByStatusId[statusId]

            val requestedPlannedDate =
                requestedItem.plannedDate.toDatabaseLocalDateTime()

            if (existingEntity == null) {
                /*
                 * Записи с таким статусом у агента ещё нет —
                 * создаём новую.
                 */
                val newEntity = AgentStatusSlaEntity().apply {
                    /*
                     * Сеттер сам заполнит:
                     * primaryKey.aiAgentId = agent.id.
                     */
                    aiAgent = agent

                    /*
                     * Сеттер сам заполнит:
                     * primaryKey.agentStatusId = statusEntity.id.
                     */
                    agentStatus = statusEntity

                    plannedDate = requestedPlannedDate

                    /*
                     * completedDate не устанавливаем.
                     * У новой записи он останется null.
                     */
                }

                entitiesToSave.add(newEntity)
                delta.add(requestedItem)

                return@forEach
            }

            /*
             * По документации сравнивается только дата,
             * временная часть не учитывается.
             */
            val plannedDateChanged =
                existingEntity.plannedDate?.toLocalDate() !=
                    requestedPlannedDate.toLocalDate()

            if (plannedDateChanged) {
                /*
                 * Меняем только plannedDate.
                 * completedDate не изменяется и не очищается.
                 */
                existingEntity.plannedDate = requestedPlannedDate

                entitiesToSave.add(existingEntity)
                delta.add(requestedItem)
            }
        }

        /*
         * Находим существующие сроки этапов,
         * которых нет в новом запросе.
         */
        val entitiesToDelete = existingStatusSla.filter { existingEntity ->
            existingEntity.primaryKey.agentStatusId !in requestedStatusIds
        }

        /*
         * Все новые и изменённые сущности сохраняются
         * одним вызовом saveAll().
         */
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
        /*
         * Метод вызывается только когда ключ statusSla
         * присутствует в PATCH-запросе.
         *
         * Поэтому null означает очистку текущего списка.
         */
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

            val plannedDate = parsePlannedDate(
                value = item[PLANNED_DATE_FIELD],
            )

            StatusSlaDto(
                status = status,
                plannedDate = plannedDate,
            )
        }
    }

    private fun parsePlannedDate(
        value: Any?,
    ): OffsetDateTime {
        return when (value) {
            is OffsetDateTime -> value

            is String -> {
                if (value.isBlank()) {
                    throwWrongStatusSlaValue()
                }

                try {
                    OffsetDateTime.parse(value)
                } catch (exception: DateTimeParseException) {
                    throwWrongStatusSlaValue()
                }
            }

            else -> throwWrongStatusSlaValue()
        }
    }

    /**
     * Техническая защита от двух сроков для одного статуса.
     *
     * Без неё обе записи будут иметь одинаковый составной ключ:
     * ai_agent_id + agent_status_id.
     */
    private fun validateDuplicateStatuses(
        requestedStatusSla: List<StatusSlaDto>,
    ) {
        val duplicateStatuses = requestedStatusSla
            .groupingBy { statusSla -> statusSla.status }
            .eachCount()
            .filterValues { count -> count > 1 }
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

    /**
     * Проверяет, что каждый code из запроса найден
     * среди активных записей справочника status.
     */
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

    /**
     * Entity хранит LocalDateTime, поэтому offset приводим к UTC,
     * после чего убираем информацию о зоне.
     */
    private fun OffsetDateTime.toDatabaseLocalDateTime(): LocalDateTime {
        return withOffsetSameInstant(ZoneOffset.UTC)
            .toLocalDateTime()
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
