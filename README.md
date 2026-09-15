```java
/**
 * Одно действие из истории изменения применимости метрики.
 */
data class MetricApplicabilityHistoryItemResponse(
    val requestId: Long,
    val action: String,
    val actorName: String,
    val comment: String?,
    val resumePeriod: LocalDate?,
    val createdAt: LocalDate
)

/**
 * Ответ с текущим состоянием применимости метрики
 * и полной историей действий по ней.
 */
data class MetricApplicabilityHistoryResponse(
    val currentApplicabilityStatus: MetricApplicabilityStatus,
    val history: List<MetricApplicabilityHistoryItemResponse>
)

/**
 * Преобразует техническое действие истории
 * в человекочитаемое значение для API.
 */
@Component
class MetricApplicabilityHistoryActionResolver {

    /**
     * Возвращает отображаемое название действия.
     */
    fun getActionName(action: MetricApplicabilityAction): String =
        when (action) {
            MetricApplicabilityAction.REQUEST_CREATED -> "Запрос на неактуальную метрику"
            MetricApplicabilityAction.APPROVED -> "Согласовано"
            MetricApplicabilityAction.REJECTED -> "Отклонено"
            MetricApplicabilityAction.CANCEL_DECISION -> "Отмена решения"
            MetricApplicabilityAction.RESTORED -> "Заявка отменена"
            MetricApplicabilityAction.RESUME_PERIOD -> "Срок отвязки метрики истёк"
        }
}

/**
 * Репозиторий состояния применимости метрик инициативы.
 */
@Repository
interface InitiativeMetricAssignmentRepository : JpaRepository<InitiativeMetricAssignmentEntity, Long> {

    /**
     * Находит assignment конкретной метрики для выбранного типа агента.
     */
    fun findByInitiativeMetricTypeIdAndMetricId(
        initiativeMetricTypeId: Long,
        metricId: UUID
    ): InitiativeMetricAssignmentEntity?
}

/**
 * Репозиторий истории изменения применимости метрик.
 */
@Repository
interface MetricApplicabilityHistoryRepository : JpaRepository<MetricApplicabilityHistoryEntity, Long> {

    /**
     * Возвращает полную историю всех заявок указанного assignment.
     *
     * В выборку входят в том числе заявки с isVisibleInOffice=false.
     * id используется как дополнительная стабильная сортировка,
     * поскольку createdAt хранится с точностью до даты.
     */
    @Query(
        """
        select history
        from MetricApplicabilityHistoryEntity history
        join fetch history.metricApplicabilityRequest request
        where request.initiativeMetricAssignment.id = :assignmentId
        order by history.createdAt asc, history.id asc
        """
    )
    fun findAllByAssignmentId(@Param("assignmentId") assignmentId: Long): List<MetricApplicabilityHistoryEntity>
}

/**
 * Сервис получения истории изменения применимости метрики.
 *
 * Отвечает за получение текущего состояния assignment,
 * полной истории связанных заявок и ФИО пользователей,
 * выполнявших действия.
 */
@Service
class MetricApplicabilityHistoryQueryService(
    private val messageProvider: MessageProvider,
    private val aiAgentRepository: AIAgentRepository,
    private val initiativeMetricTypeRepository: InitiativeMetricTypeRepository,
    private val initiativeMetricAssignmentRepository: InitiativeMetricAssignmentRepository,
    private val metricsDirectoryRepository: MetricsDirectoryRepository,
    private val metricApplicabilityHistoryRepository: MetricApplicabilityHistoryRepository,
    private val metricApplicabilityHistoryActionResolver: MetricApplicabilityHistoryActionResolver,
    private val userAccountService: UserAccountService
) {

    companion object {
        private const val SYSTEM_ACTOR = "SYSTEM"
    }

    /**
     * Возвращает текущий статус применимости метрики и полную историю действий.
     *
     * Если assignment ещё не создавался, метрика считается ACTIVE,
     * а история возвращается пустой.
     */
    @Transactional(readOnly = true)
    fun getApplicabilityHistory(
        initiativeId: Long,
        metricId: UUID,
        agentType: String
    ): MetricApplicabilityHistoryResponse {
        validateInitiativeExists(initiativeId)
        validateMetricExists(metricId)

        val initiativeMetricType = getInitiativeMetricType(initiativeId, agentType)

        val initiativeMetricTypeId = initiativeMetricType.id
            ?: error("У сохранённого типа агента инициативы отсутствует id")

        val assignment = initiativeMetricAssignmentRepository.findByInitiativeMetricTypeIdAndMetricId(
            initiativeMetricTypeId = initiativeMetricTypeId,
            metricId = metricId
        )

        if (assignment == null) {
            return MetricApplicabilityHistoryResponse(
                currentApplicabilityStatus = MetricApplicabilityStatus.ACTIVE,
                history = emptyList()
            )
        }

        val assignmentId = assignment.id ?: error("У сохранённого assignment отсутствует id")
        val historyEntries = metricApplicabilityHistoryRepository.findAllByAssignmentId(assignmentId)

        val actorUserIds = getActorUserIds(historyEntries)
        val usersById = userAccountService.getUsersByIds(actorUserIds)

        return MetricApplicabilityHistoryResponse(
            currentApplicabilityStatus = assignment.applicabilityStatus,
            history = historyEntries.map { historyEntry -> historyEntry.toResponse(usersById) }
        )
    }

    /**
     * Проверяет существование инициативы.
     */
    private fun validateInitiativeExists(initiativeId: Long) {
        aiAgentRepository.findByIdOrNull(initiativeId)
            ?: throw AiBadRequestException(
                errorCode = INITIATIVE_NOT_FOUND,
                message = MessageFormat.format(messageProvider[INITIATIVE_NOT_FOUND], initiativeId)
            )
    }

    /**
     * Проверяет существование метрики.
     */
    private fun validateMetricExists(metricId: UUID) {
        metricsDirectoryRepository.findByIdOrNull(metricId)
            ?: throw AiBadRequestException(
                errorCode = INITIATIVE_METRIC_NOT_FOUND,
                message = MessageFormat.format(messageProvider[INITIATIVE_METRIC_NOT_FOUND], metricId)
            )
    }

    /**
     * Возвращает тип агента, принадлежащий указанной инициативе.
     */
    private fun getInitiativeMetricType(
        initiativeId: Long,
        agentType: String
    ): InitiativeMetricTypeEntity =
        initiativeMetricTypeRepository.findByAiAgentIdAndAgentType(initiativeId, agentType)
            ?: throw AiBadRequestException(
                errorCode = INITIATIVE_METRIC_AGENT_TYPE_NOT_FOUND,
                message = MessageFormat.format(
                    messageProvider[INITIATIVE_METRIC_AGENT_TYPE_NOT_FOUND],
                    agentType,
                    initiativeId
                )
            )

    /**
     * Собирает уникальные идентификаторы пользователей,
     * выполнявших действия с заявками.
     *
     * SYSTEM является техническим автором и в prm-auth не отправляется.
     */
    private fun getActorUserIds(historyEntries: List<MetricApplicabilityHistoryEntity>): Set<Long> =
        historyEntries
            .filter { historyEntry -> historyEntry.createdBy != SYSTEM_ACTOR }
            .mapNotNull { historyEntry -> historyEntry.createdBy.toLongOrNull() }
            .toSet()

    /**
     * Преобразует запись истории в DTO ответа.
     */
    private fun MetricApplicabilityHistoryEntity.toResponse(
        usersById: Map<Long, UserAccountDto>
    ): MetricApplicabilityHistoryItemResponse {
        val request = metricApplicabilityRequest

        return MetricApplicabilityHistoryItemResponse(
            requestId = request.id ?: error("У сохранённой заявки отсутствует id"),
            action = metricApplicabilityHistoryActionResolver.getActionName(action),
            actorName = getActorName(createdBy, usersById),
            comment = comment,
            resumePeriod = if (action == MetricApplicabilityAction.APPROVED) request.resumePeriod else null,
            createdAt = createdAt
        )
    }

    /**
     * Возвращает ФИО пользователя, выполнившего действие.
     *
     * Для системного действия возвращается SYSTEM.
     * Если пользователь не найден в prm-auth, возвращается его идентификатор.
     */
    private fun getActorName(
        createdBy: String,
        usersById: Map<Long, UserAccountDto>
    ): String {
        if (createdBy == SYSTEM_ACTOR) {
            return SYSTEM_ACTOR
        }

        val actorUserId = createdBy.toLongOrNull() ?: return createdBy

        return usersById[actorUserId]
            ?.let(userAccountService::getFullName)
            ?: createdBy
    }
}

/**
 * REST API получения истории изменения применимости метрик.
 */
@Validated
@RestController
@RequestMapping("/api/v1/ai-agent")
class MetricApplicabilityHistoryController(
    private val metricApplicabilityHistoryQueryService: MetricApplicabilityHistoryQueryService
) {

    /**
     * Возвращает историю действий по конкретной метрике,
     * инициативе и типу агента.
     */
    @PreAuthorize("hasAnyAuthority('PROJECT_OFFICE', 'CMS_ADMIN', 'TRANSFORMATION_OFFICE')")
    @GetMapping("/initiatives/{initiativeId}/metrics/{metricId}/{agentType}/applicability-history")
    fun getApplicabilityHistory(
        @PathVariable @Positive initiativeId: Long,
        @PathVariable metricId: UUID,
        @PathVariable @NotBlank agentType: String
    ): MetricApplicabilityHistoryResponse =
        metricApplicabilityHistoryQueryService.getApplicabilityHistory(
            initiativeId = initiativeId,
            metricId = metricId,
            agentType = agentType
        )
}
```
