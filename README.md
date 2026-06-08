```java

@Component
class StatusSlaUpdater(
    private val agentStatusSlaRepository: AgentStatusSlaRepository,
    private val statusRepository: StatusRepository,
    private val messageProvider: MessageProvider,
) {

    /**
     * Синхронизирует записи agent_status_sla с данными из PATCH-запроса.
     *
     * Возвращает дельту:
     * - новые записи;
     * - существующие записи, у которых изменилась plannedDate.
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
         * До окончания валидации данные в БД не изменяются.
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
         * Если переданы null или пустой список,
         * удаляем все SLA агента.
         */
        if (requestedStatusSla.isEmpty()) {
            if (existingStatusSla.isNotEmpty()) {
                agentStatusSlaRepository.deleteAll(existingStatusSla)
            }

            return emptyList()
        }

        /*
         * После parseStatusSla status гарантированно не null,
         * поэтому здесь используем requireNotNull.
         */
        val requestedStatusCodes: Set<String> = requestedStatusSla
            .map { statusSla ->
                requireNotNull(statusSla.status)
            }
            .toSet()

        /*
         * Одним запросом получаем активные статусы справочника.
         */
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

        /*
         * Текущие SLA индексируем по ID статуса.
         */
        val existingStatusSlaByStatusId = existingStatusSla
            .associateBy { entity ->
                entity.primaryKey.agentStatusId
            }

        val requestedStatusIds = mutableSetOf<Long>()

        /*
         * Сюда собираем и новые, и изменённые сущности,
         * чтобы выполнить один saveAll().
         */
        val entitiesToSave = mutableListOf<AgentStatusSlaEntity>()

        /*
         * Дельта для jira_change с changeType = plannedDate.
         */
        val delta = mutableListOf<StatusSlaDto>()

        requestedStatusSla.forEach { requestedItem ->
            val statusCode = requireNotNull(requestedItem.status)
            val plannedDate = requireNotNull(requestedItem.plannedDate)

            val statusEntity = requireNotNull(
                activeStatusesByCode[statusCode]
            )

            val statusId = requireNotNull(statusEntity.id)

            requestedStatusIds.add(statusId)

            val existingEntity =
                existingStatusSlaByStatusId[statusId]

            /*
             * Entity хранит LocalDateTime,
             * DTO содержит LocalDate.
             */
            val requestedPlannedDateTime =
                plannedDate.atStartOfDay()

            if (existingEntity == null) {
                val newEntity = AgentStatusSlaEntity().apply {
                    /*
                     * Сеттер заполнит primaryKey.aiAgentId.
                     */
                    aiAgent = agent

                    /*
                     * Сеттер заполнит primaryKey.agentStatusId.
                     */
                    agentStatus = statusEntity

                    plannedDate = requestedPlannedDateTime

                    /*
                     * completedDate для новой записи не заполняем.
                     */
                }

                entitiesToSave.add(newEntity)

                /*
                 * В дельту completedDate не передаём,
                 * потому что JIRA-синхронизация касается plannedDate.
                 */
                delta.add(
                    StatusSlaDto(
                        status = statusCode,
                        plannedDate = plannedDate,
                    )
                )

                return@forEach
            }

            /*
             * Сравнение только по календарной дате.
             *
             * Хотя DTO уже содержит LocalDate,
             * entity хранит LocalDateTime, поэтому у старого значения
             * берём только toLocalDate().
             */
            val plannedDateChanged =
                existingEntity.plannedDate?.toLocalDate() != plannedDate

            if (plannedDateChanged) {
                /*
                 * Меняем только plannedDate.
                 * completedDate не трогаем.
                 */
                existingEntity.plannedDate =
                    requestedPlannedDateTime

                entitiesToSave.add(existingEntity)

                delta.add(
                    StatusSlaDto(
                        status = statusCode,
                        plannedDate = plannedDate,
                    )
                )
            }
        }

        /*
         * Удаляем записи для этапов,
         * отсутствующих в новом запросе.
         */
        val entitiesToDelete = existingStatusSla.filter { existingEntity ->
            existingEntity.primaryKey.agentStatusId !in requestedStatusIds
        }

        /*
         * Новые и изменённые сущности сохраняем одним вызовом.
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
         * Метод вызывается только при наличии ключа statusSla.
         * null интерпретируется как очистка списка.
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
                    /*
                     * Поддерживает строку формата:
                     * 2026-05-03
                     */
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
            .groupingBy { statusCode -> statusCode }
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
