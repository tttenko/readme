```java

/**
 * Формирует записи об изменениях AI-агента, которые необходимо
 * синхронизировать с JIRA.
 *
 * Синхронизация выполняется только для агентов, связанных с инициативой
 * проекта CROSSGOAL через таблицу [JiraIssueEntity].
 *
 * Компонент может создать два типа записей в таблице `jira_change`:
 *
 * - `initiative` — изменение основных параметров инициативы;
 * - `plannedDate` — изменение плановых сроков этапов реализации.
 *
 * Если была создана хотя бы одна запись `jira_change`, агенту выставляется
 * статус `pendingUpdate`.
 *
 * Компонент не отправляет данные непосредственно в JIRA. Он только сохраняет
 * изменения, которые впоследствии должны быть обработаны механизмом синхронизации.
 */
@Component
open class JiraChangeUpdater(

    /**
     * Репозиторий JIRA-задач.
     *
     * Используется для проверки наличия у агента инициативы,
     * связанной с проектом CROSSGOAL.
     */
    private val jiraIssueRepository: JiraIssueRepository,

    /**
     * Репозиторий изменений, ожидающих синхронизации с JIRA.
     */
    private val jiraChangeRepository: JiraChangeRepository,

    /**
     * Jackson-мэппер, используемый для преобразования payload
     * из объектов и коллекций Kotlin в JSON-представление.
     */
    private val objectMapper: ObjectMapper,
) {

    /**
     * Определяет необходимость синхронизации изменений агента с JIRA
     * и создаёт соответствующие записи в таблице `jira_change`.
     *
     * Сначала проверяется, связан ли агент с инициативой проекта CROSSGOAL.
     * Если связи нет, метод завершает работу без создания записей.
     *
     * При наличии связи могут быть созданы:
     *
     * - изменение типа `initiative`, если PATCH-запрос содержит хотя бы одно
     *   поле из списка [INITIATIVE_SYNC_FIELDS];
     * - изменение типа `plannedDate`, если [changedStatusSla] содержит новые
     *   или изменённые плановые сроки.
     *
     * Если создана хотя бы одна запись, агент переводится в статус
     * `pendingUpdate`.
     *
     * @param agent обновляемый AI-агент.
     * @param request исходные параметры PATCH-запроса. В payload попадут
     * только поля, разрешённые для синхронизации с JIRA.
     * @param changedStatusSla дельта плановых сроков, сформированная
     * `StatusSlaUpdater`. Содержит только новые сроки и сроки,
     * у которых изменилась плановая дата.
     * @param userId идентификатор пользователя, выполнившего обновление.
     */
    open fun update(
        agent: AIAgentEntity,
        request: Map<String, Any?>,
        changedStatusSla: List<StatusSlaDto>,
        userId: Long,
    ) {
        /**
         * Идентификатор обновляемого агента.
         *
         * Агент к этому моменту уже должен быть сохранён в БД,
         * поэтому его ID используется для поиска связанной JIRA-инициативы.
         */
        val agentId = agent.id

        /**
         * Признак наличия у агента инициативы проекта CROSSGOAL.
         *
         * Если такой записи в `jira_issue` нет, изменения агента
         * не требуется синхронизировать с JIRA.
         */
        val hasCrossgoalInitiative =
            jiraIssueRepository.existsByAgentIdAndTypeAndProject(
                agentId = agentId,
                type = INITIATIVE_ISSUE_TYPE,
                project = CROSSGOAL_PROJECT,
            )

        if (!hasCrossgoalInitiative) {
            return
        }

        /**
         * Единое время выполнения операции.
         *
         * Используется как дата создания всех сформированных `jira_change`
         * и как дата последнего обновления агента.
         */
        val currentDateTime = LocalDateTime.now()

        /**
         * Записи об изменениях, которые будут сохранены в `jira_change`.
         *
         * В рамках одного PATCH-запроса список может содержать:
         *
         * - только `initiative`;
         * - только `plannedDate`;
         * - обе записи;
         * - ни одной записи.
         */
        val jiraChangesToSave =
            mutableListOf<JiraChangeEntity>()

        createInitiativeChange(
            agent = agent,
            request = request,
            created = currentDateTime,
        )?.let { initiativeChange ->
            jiraChangesToSave.add(initiativeChange)
        }

        createPlannedDateChange(
            agent = agent,
            changedStatusSla = changedStatusSla,
            created = currentDateTime,
        )?.let { plannedDateChange ->
            jiraChangesToSave.add(plannedDateChange)
        }

        if (jiraChangesToSave.isEmpty()) {
            return
        }

        jiraChangeRepository.saveAll(
            jiraChangesToSave
        )

        agent.jiraStatus = PENDING_UPDATE_STATUS
        agent.updated = currentDateTime
        agent.updatedBy = userId
    }

    /**
     * Формирует запись об изменении основных параметров инициативы.
     *
     * Из исходного PATCH-запроса выбираются только поля, перечисленные
     * в [INITIATIVE_SYNC_FIELDS]. Остальные параметры запроса
     * в payload не попадают.
     *
     * Наличие ключа учитывается независимо от его значения. Поэтому поле,
     * явно переданное со значением `null`, также попадёт в payload.
     *
     * @param agent агент, для которого создаётся запись изменения.
     * @param request исходные параметры PATCH-запроса.
     * @param created дата и время создания записи.
     *
     * @return сущность изменения типа `initiative` либо `null`,
     * если запрос не содержит полей, требующих синхронизации.
     */
    private fun createInitiativeChange(
        agent: AIAgentEntity,
        request: Map<String, Any?>,
        created: LocalDateTime,
    ): JiraChangeEntity? {

        /**
         * Часть PATCH-запроса, содержащая только параметры,
         * подлежащие синхронизации с инициативой в JIRA.
         */
        val initiativePayload = request
            .filterKeys { fieldName ->
                fieldName in INITIATIVE_SYNC_FIELDS
            }

        if (initiativePayload.isEmpty()) {
            return null
        }

        return JiraChangeEntity(
            agent = agent,
            changeType = INITIATIVE_CHANGE_TYPE,
            payload = objectMapper.valueToTree(
                initiativePayload
            ),
            created = created,
        )
    }

    /**
     * Формирует запись об изменении плановых сроков этапов реализации.
     *
     * Метод получает уже рассчитанную дельту от `StatusSlaUpdater`,
     * поэтому повторное сравнение текущих и запрошенных дат не выполняется.
     *
     * Каждая плановая дата преобразуется из [LocalDate] в момент начала дня
     * по UTC для формирования даты-времени в JIRA payload.
     *
     * @param agent агент, для которого создаётся запись изменения.
     * @param changedStatusSla только новые или изменённые плановые сроки.
     * @param created дата и время создания записи.
     *
     * @return сущность изменения типа `plannedDate` либо `null`,
     * если плановые сроки не изменялись.
     */
    private fun createPlannedDateChange(
        agent: AIAgentEntity,
        changedStatusSla: List<StatusSlaDto>,
        created: LocalDateTime,
    ): JiraChangeEntity? {

        if (changedStatusSla.isEmpty()) {
            return null
        }

        /**
         * Изменённые плановые сроки в формате, используемом
         * для формирования JIRA payload.
         */
        val changedPlannedDates =
            changedStatusSla.map { statusSla ->

                /**
                 * Код этапа реализации, например `pilot`
                 * или `targetSolution`.
                 */
                val status =
                    requireNotNull(statusSla.status)

                /**
                 * Новая плановая дата этапа без временной составляющей.
                 */
                val plannedDate =
                    requireNotNull(statusSla.plannedDate)

                JiraPlannedDateDto(
                    status = status,
                    plannedDate = plannedDate
                        .atStartOfDay()
                        .toInstant(ZoneOffset.UTC),
                )
            }

        /**
         * Итоговый JSON-объект с блоком `statusSla`,
         * который будет сохранён в `jira_change.payload`.
         */
        val plannedDatePayload =
            JiraPlannedDatePayload(
                statusSla = changedPlannedDates,
            )

        return JiraChangeEntity(
            agent = agent,
            changeType = PLANNED_DATE_CHANGE_TYPE,
            payload = objectMapper.valueToTree(
                plannedDatePayload
            ),
            created = created,
        )
    }

    private companion object {

        /**
         * Код проекта JIRA, наличие инициативы которого означает,
         * что изменения агента должны синхронизироваться с JIRA.
         */
        const val CROSSGOAL_PROJECT = "crossgoal"

        /**
         * Тип JIRA-задачи, представляющей инициативу агента.
         */
        const val INITIATIVE_ISSUE_TYPE = "initiative"

        /**
         * Тип записи `jira_change` для изменений основных
         * параметров инициативы.
         */
        const val INITIATIVE_CHANGE_TYPE = "initiative"

        /**
         * Тип записи `jira_change` для изменений плановых сроков.
         */
        const val PLANNED_DATE_CHANGE_TYPE = "plannedDate"

        /**
         * Статус агента, означающий, что его изменения ожидают
         * отправки из Пульта в JIRA.
         */
        const val PENDING_UPDATE_STATUS = "pendingUpdate"

        /**
         * Поля PATCH-запроса, изменения которых должны передаваться
         * в JIRA как изменение инициативы.
         *
         * Поля запроса, отсутствующие в этом наборе, не включаются
         * в `jira_change.payload`.
         */
        val INITIATIVE_SYNC_FIELDS = setOf(
            "agentName",
            "agentDescription",
            "agentInitiativeType",
            "block",
            "division",
            "strategies",
            "processes",
            "enablers",
            "involvedResources",
            "agentEffectOptimization",
            "agentEffectRevenue",
        )
    }
}

/**
 * Обновляет плановые сроки этапов реализации AI-агента.
 *
 * Компонент синхронизирует переданный блок `statusSla`
 * с записями таблицы `agent_status_sla`:
 *
 * - создаёт новые плановые сроки;
 * - обновляет изменившиеся плановые даты;
 * - удаляет сроки, отсутствующие в новом запросе;
 * - не изменяет фактическую дату завершения `completedDate`;
 * - возвращает только новые и фактически изменённые сроки.
 *
 * Возвращаемая дельта используется для создания записи
 * `jira_change` с типом `plannedDate`.
 *
 * При сравнении плановых сроков учитывается только календарная дата.
 * Временная составляющая существующего значения не сравнивается.
 */
@Component
open class StatusSlaUpdater(

    /**
     * Репозиторий плановых и фактических сроков этапов агента.
     *
     * Используется для получения, сохранения и удаления записей
     * таблицы `agent_status_sla`.
     */
    private val agentStatusSlaRepository: AgentStatusSlaRepository,

    /**
     * Репозиторий справочника статусов агента.
     *
     * Используется для проверки существования переданных
     * в запросе кодов статусов.
     */
    private val statusRepository: StatusRepository,

    /**
     * Провайдер локализованных сообщений об ошибках.
     */
    private val messageProvider: MessageProvider,
) {

    /**
     * Синхронизирует плановые сроки агента с блоком `statusSla`
     * из PATCH-запроса.
     *
     * Алгоритм работы:
     *
     * 1. Преобразует необработанное значение запроса в список [StatusSlaDto].
     * 2. Получает существующие сроки агента из БД.
     * 3. Проверяет, что все переданные статусы существуют
     *    в активном справочнике статусов.
     * 4. Создаёт новые записи и обновляет изменившиеся плановые даты.
     * 5. Удаляет записи для статусов, отсутствующих в запросе.
     * 6. Возвращает только новые и изменённые сроки для синхронизации с JIRA.
     *
     * Если [rawStatusSlaValue] равен `null` или содержит пустой список,
     * все существующие плановые сроки агента удаляются.
     *
     * Фактическая дата завершения `completedDate` существующих записей
     * не изменяется.
     *
     * @param agent агент, для которого обновляются плановые сроки.
     * @param rawStatusSlaValue необработанное значение поля `statusSla`
     * из PATCH-запроса. Ожидается список объектов с полями
     * `status` и `plannedDate`.
     *
     * @return список новых и изменённых плановых сроков.
     * Неизменившиеся и удалённые сроки в результат не включаются.
     *
     * @throws AiBadRequestException если структура `statusSla`
     * некорректна, дата имеет неверный формат или переданный статус
     * отсутствует в активном справочнике.
     */
    open fun update(
        agent: AIAgentEntity,
        rawStatusSlaValue: Any?,
    ): List<StatusSlaDto> {

        /**
         * Идентификатор агента, используемый для получения
         * существующих записей `agent_status_sla`.
         */
        val agentId = agent.id

        /**
         * Плановые сроки, полученные из PATCH-запроса
         * и преобразованные в типизированные DTO.
         */
        val requestedStatusSlaItems = parseStatusSla(
            rawStatusSla = rawStatusSlaValue,
        )

        /**
         * Существующие в БД записи плановых сроков агента
         * до применения текущего PATCH-запроса.
         */
        val persistedStatusSlaEntities =
            agentStatusSlaRepository.findAllByAiAgentId(
                agentId = agentId,
            )

        /*
         * Пустой список или null означает удаление
         * всех плановых сроков агента.
         */
        if (requestedStatusSlaItems.isEmpty()) {
            if (persistedStatusSlaEntities.isNotEmpty()) {
                agentStatusSlaRepository.deleteAll(
                    persistedStatusSlaEntities
                )
            }

            return emptyList()
        }

        /**
         * Уникальные коды статусов, переданные в запросе.
         *
         * Используются для проверки существования статусов
         * в справочнике.
         */
        val requestedStatusCodes = requestedStatusSlaItems
            .map { requestedStatusSla ->
                requireNotNull(requestedStatusSla.status)
            }
            .toSet()

        /**
         * Активные статусы, сгруппированные по их коду.
         *
         * Ключом является код статуса, значением — сущность справочника.
         * Отключённые статусы в карту не включаются.
         */
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

        /**
         * Существующие сроки агента, сгруппированные по ID статуса.
         *
         * Позволяет за константное время определить,
         * существует ли уже SLA для конкретного этапа.
         */
        val persistedStatusSlaByStatusId =
            persistedStatusSlaEntities.associateBy { persistedStatusSla ->
                persistedStatusSla.primaryKey.agentStatusId
            }

        /**
         * ID статусов, присутствующих в текущем запросе.
         *
         * После обработки используется для определения записей,
         * которые необходимо удалить из БД.
         */
        val requestedStatusIds = mutableSetOf<Long>()

        /**
         * Новые и изменённые сущности `agent_status_sla`,
         * которые необходимо сохранить.
         */
        val statusSlaEntitiesToSave =
            mutableListOf<AgentStatusSlaEntity>()

        /**
         * Новые и фактически изменённые плановые сроки.
         *
         * Этот список является дельтой изменений и впоследствии
         * используется для создания `jira_change`
         * с типом `plannedDate`.
         */
        val changedStatusSlaItems =
            mutableListOf<StatusSlaDto>()

        requestedStatusSlaItems.forEach { requestedStatusSla ->

            /**
             * Код этапа реализации, переданный в запросе.
             */
            val requestedStatusCode =
                requireNotNull(requestedStatusSla.status)

            /**
             * Плановая дата этапа, переданная в запросе.
             */
            val requestedPlannedDate =
                requireNotNull(requestedStatusSla.plannedDate)

            /**
             * Сущность активного статуса, найденная по коду
             * из запроса.
             */
            val requestedStatusEntity = requireNotNull(
                activeStatusesByCode[requestedStatusCode]
            )

            /**
             * Идентификатор статуса, используемый в составном ключе
             * таблицы `agent_status_sla`.
             */
            val requestedStatusId =
                requireNotNull(requestedStatusEntity.id)

            requestedStatusIds.add(requestedStatusId)

            /**
             * Существующая запись SLA для текущего статуса.
             *
             * Значение равно `null`, если срок для этого этапа
             * ранее не создавался.
             */
            val persistedStatusSla =
                persistedStatusSlaByStatusId[requestedStatusId]

            /**
             * Плановая дата, преобразованная в дату-время начала дня.
             *
             * В таблице дата хранится как [LocalDateTime],
             * хотя при сравнении учитывается только [LocalDate].
             */
            val requestedPlannedDateTime =
                requestedPlannedDate.atStartOfDay()

            /*
             * Такой SLA ещё не существовал.
             */
            if (persistedStatusSla == null) {
                /**
                 * Новая запись планового срока для агента и статуса.
                 */
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

            /**
             * Признак изменения плановой даты.
             *
             * При сравнении учитывается только календарная дата,
             * временная составляющая существующего значения игнорируется.
             */
            val hasPlannedDateChanged =
                persistedStatusSla.plannedDate?.toLocalDate() !=
                    requestedPlannedDate

            if (hasPlannedDateChanged) {
                /*
                 * Меняется только plannedDate.
                 * completedDate остаётся без изменений.
                 */
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

        /**
         * Существующие сроки, статусы которых отсутствуют
         * в новом PATCH-запросе.
         *
         * Эти записи должны быть удалены из `agent_status_sla`.
         */
        val statusSlaEntitiesToDelete =
            persistedStatusSlaEntities.filter { persistedStatusSla ->
                persistedStatusSla.primaryKey.agentStatusId !in
                    requestedStatusIds
            }

        if (statusSlaEntitiesToSave.isNotEmpty()) {
            agentStatusSlaRepository.saveAll(
                statusSlaEntitiesToSave
            )
        }

        if (statusSlaEntitiesToDelete.isNotEmpty()) {
            agentStatusSlaRepository.deleteAll(
                statusSlaEntitiesToDelete
            )
        }

        return changedStatusSlaItems
    }

    /**
     * Преобразует необработанное значение `statusSla`
     * из PATCH-запроса в список [StatusSlaDto].
     *
     * Ожидаемый формат каждого элемента:
     *
     * ```json
     * {
     *   "status": "pilot",
     *   "plannedDate": "2026-05-03"
     * }
     * ```
     *
     * Значение `null` интерпретируется как пустой список,
     * то есть как запрос на удаление всех плановых сроков.
     *
     * @param rawStatusSla необработанное значение поля `statusSla`.
     *
     * @return типизированный список плановых сроков.
     *
     * @throws AiBadRequestException если значение не является списком,
     * элемент списка не является объектом, код статуса пустой
     * или плановая дата имеет недопустимое значение.
     */
    private fun parseStatusSla(
        rawStatusSla: Any?,
    ): List<StatusSlaDto> {

        if (rawStatusSla == null) {
            return emptyList()
        }

        /**
         * Необработанный список SLA из тела PATCH-запроса.
         */
        val statusSlaList = rawStatusSla as? List<*>
            ?: throwWrongStatusSlaValue()

        return statusSlaList.map { rawItem ->

            /**
             * Один элемент списка `statusSla`,
             * представленный JSON-объектом.
             */
            val item = rawItem as? Map<*, *>
                ?: throwWrongStatusSlaValue()

            /**
             * Код этапа реализации из поля `status`.
             */
            val status = item[STATUS_FIELD] as? String

            if (status.isNullOrBlank()) {
                throwWrongStatusSlaValue()
            }

            /**
             * Плановая дата этапа, преобразованная в [LocalDate].
             */
            val plannedDate = parseLocalDate(
                value = item[PLANNED_DATE_FIELD],
            )

            StatusSlaDto(
                status = status,
                plannedDate = plannedDate,
            )
        }
    }

    /**
     * Преобразует значение плановой даты в [LocalDate].
     *
     * Поддерживаются:
     *
     * - готовый объект [LocalDate];
     * - строка в стандартном ISO-формате `yyyy-MM-dd`.
     *
     * @param value необработанное значение поля `plannedDate`.
     *
     * @return преобразованная календарная дата.
     *
     * @throws AiBadRequestException если значение отсутствует,
     * имеет неподдерживаемый тип, является пустой строкой
     * или не соответствует формату ISO Local Date.
     */
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

    /**
     * Проверяет существование всех переданных кодов статусов
     * в активном справочнике.
     *
     * Отключённый статус считается отсутствующим, поскольку карта
     * [activeStatusesByCode] содержит только активные значения.
     *
     * @param requestedStatusCodes уникальные коды статусов
     * из PATCH-запроса.
     * @param activeStatusesByCode активные статусы,
     * сгруппированные по коду.
     *
     * @throws AiBadRequestException если хотя бы один код
     * отсутствует в активном справочнике.
     */
    private fun validateRequestedStatusesExist(
        requestedStatusCodes: Set<String>,
        activeStatusesByCode: Map<String, StatusEntity>,
    ) {
        /**
         * Коды статусов из запроса, для которых не найдено
         * активного значения в справочнике.
         */
        val missingStatusCodes =
            requestedStatusCodes.filterNot { requestedStatusCode ->
                activeStatusesByCode.containsKey(requestedStatusCode)
            }

        if (missingStatusCodes.isNotEmpty()) {
            throw AiBadRequestException(
                errorCode = STATUS_WITH_CODE_NOT_FOUND,
                message = MessageFormat.format(
                    messageProvider[STATUS_WITH_CODE_NOT_FOUND],
                    missingStatusCodes.joinToString(),
                ),
            )
        }
    }

    /**
     * Завершает обработку ошибкой некорректного значения `statusSla`.
     *
     * Метод имеет тип возврата [Nothing], поскольку всегда
     * выбрасывает исключение и не возвращает управление вызывающему коду.
     *
     * @throws AiBadRequestException всегда.
     */
    private fun throwWrongStatusSlaValue(): Nothing {
        throw AiBadRequestException(
            errorCode = WRONG_STATUS_SLA_VALUE,
            message = messageProvider[WRONG_STATUS_SLA_VALUE],
        )
    }

    private companion object {

        /**
         * Имя поля с кодом этапа внутри элемента `statusSla`.
         */
        const val STATUS_FIELD = "status"

        /**
         * Имя поля с плановой датой внутри элемента `statusSla`.
         */
        const val PLANNED_DATE_FIELD = "plannedDate"
    }
}

```
