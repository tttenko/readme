```java
@Service
class MetricApplicabilityRequestUpdater(
    private val metricApplicabilityRequestRepository:
        MetricApplicabilityRequestRepository,
    private val initiativeMetricAssignmentRepository:
        InitiativeMetricAssignmentRepository,
    private val metricApplicabilityHistoryRepository:
        MetricApplicabilityHistoryRepository,
    private val initiativeMetricTypeRepository:
        InitiativeMetricTypeRepository,
    private val metricsDirectoryRepository:
        MetricsDirectoryRepository,
    private val metricApplicabilityActionResolver:
        MetricApplicabilityActionResolver,
    private val userInfoProvider:
        UserInfoProvider,
    private val userAccountService:
        UserAccountService,
    private val emailHandler:
        EmailHandler,
    private val emailProperties:
        EmailProperties,
    private val messageProvider:
        MessageProvider
) {

    companion object {
        private val log by logger()
    }

    /**
     * Выполняет действие Офиса над заявкой
     * на неприменимость метрики.
     *
     * Поддерживает APPROVE, REJECT и CANCEL_DECISION.
     * Изменение request, assignment и history выполняется
     * в одной транзакции.
     */
    @Transactional
    fun updateRequest(
        initiativeId: Long,
        metricId: UUID,
        agentType: String,
        requestId: Long,
        request: UpdateMetricApplicabilityRequest,
    ): MetricApplicabilityRequestActionResponse {

        val currentUserId = userInfoProvider.currentUser().id

        val initiativeMetricType =
            initiativeMetricTypeRepository
                .findByAiAgentIdAndAgentType(
                    initiativeId = initiativeId,
                    agentType = agentType
                )
                ?: throw AiNotFoundException(
                    errorCode = INITIATIVE_AGENT_TYPE_NOT_FOUND,
                    message = MessageFormat.format(
                        messageProvider[INITIATIVE_AGENT_TYPE_NOT_FOUND],
                        initiativeId,
                        agentType
                    )
                )

        val initiativeMetricTypeId =
            initiativeMetricType.id
                ?: error(
                    "У сохранённого типа агента инициативы отсутствует id"
                )

        validateMetricExists(metricId)

        val assignment =
            initiativeMetricAssignmentRepository
                .findByInitiativeMetricTypeIdAndMetricId(
                    initiativeMetricTypeId = initiativeMetricTypeId,
                    metricId = metricId
                )
                ?: throw AiNotFoundException(
                    errorCode = INITIATIVE_METRIC_ASSIGNMENT_NOT_FOUND,
                    message = MessageFormat.format(
                        messageProvider[
                            INITIATIVE_METRIC_ASSIGNMENT_NOT_FOUND
                        ],
                        initiativeId,
                        metricId,
                        agentType
                    )
                )

        val assignmentId =
            assignment.id
                ?: error(
                    "У сохранённого assignment отсутствует id"
                )

        /*
         * Заявка блокируется до завершения транзакции,
         * чтобы два решения не могли выполниться одновременно.
         */
        val applicabilityRequest =
            metricApplicabilityRequestRepository
                .findByIdForUpdate(requestId)
                ?: throw AiNotFoundException(
                    errorCode = METRIC_APPLICABILITY_REQUEST_NOT_FOUND,
                    message = MessageFormat.format(
                        messageProvider[
                            METRIC_APPLICABILITY_REQUEST_NOT_FOUND
                        ],
                        requestId
                    )
                )

        validateRequestBelongsToAssignment(
            request = applicabilityRequest,
            assignmentId = assignmentId
        )

        validateAction(
            request = applicabilityRequest,
            assignment = assignment,
            action = request.action
        )

        /*
         * Валидатор уже проверил обязательность comment.
         * Здесь только убираем лишние пробелы.
         */
        val comment =
            request.comment
                ?.trim()
                ?.takeIf { it.isNotEmpty() }

        when (request.action) {

            MetricApplicabilityRequestAction.APPROVE ->
                approve(
                    request = applicabilityRequest,
                    assignment = assignment,
                    comment = comment,
                    currentUserId = currentUserId
                )

            MetricApplicabilityRequestAction.REJECT ->
                reject(
                    request = applicabilityRequest,
                    assignment = assignment,
                    comment = comment,
                    currentUserId = currentUserId
                )

            MetricApplicabilityRequestAction.CANCEL_DECISION ->
                cancelDecision(
                    request = applicabilityRequest,
                    assignment = assignment,
                    comment = comment,
                    currentUserId = currentUserId
                )
        }

        val pendingCount =
            metricApplicabilityRequestRepository
                .countByStatusAndIsVisibleInOfficeTrue(
                    MetricApplicabilityRequestStatus.PENDING
                )

        val availableActions =
            metricApplicabilityActionResolver
                .getAvailableActions(
                    requestStatus = applicabilityRequest.status,
                    applicabilityStatus =
                        assignment.applicabilityStatus
                )

        /*
         * Lazy Entity после commit не используем.
         * Поэтому необходимые данные для письма
         * достаём внутри транзакции заранее.
         */
        val metricName =
            assignment.metric.name.toString()

        val initiativeName =
            assignment
                .initiativeMetricType
                .aiAgent
                .agentName
                .toString()

        registerNotificationAfterCommit(
            recipientUserId = applicabilityRequest.createdBy,
            action = request.action,
            metricName = metricName,
            initiativeName = initiativeName
        )

        return MetricApplicabilityRequestActionResponse(
            requestId =
                applicabilityRequest.id
                    ?: error(
                        "У сохранённой заявки отсутствует id"
                    ),
            requestStatus = applicabilityRequest.status,
            applicabilityStatus =
                assignment.applicabilityStatus,
            pendingCount = pendingCount,
            availableActions = availableActions
        )
    }

    /**
     * Согласовывает неприменимость:
     * PENDING/PENDING -> APPROVED/NOT_APPLICABLE.
     */
    private fun approve(
        request: MetricApplicabilityRequestEntity,
        assignment: InitiativeMetricAssignmentEntity,
        comment: String?,
        currentUserId: Long
    ) {
        request.status =
            MetricApplicabilityRequestStatus.APPROVED

        request.decisionBy = currentUserId
        request.updatedAt = LocalDateTime.now()
        request.effectiveToPeriod = request.resumePeriod

        assignment.applicabilityStatus =
            MetricApplicabilityStatus.NOT_APPLICABLE

        saveHistory(
            request = request,
            action = MetricApplicabilityAction.APPROVED,
            currentUserId = currentUserId,
            comment = comment
        )
    }

    /**
     * Отклоняет заявку:
     * PENDING/PENDING -> REJECTED/ACTIVE.
     */
    private fun reject(
        request: MetricApplicabilityRequestEntity,
        assignment: InitiativeMetricAssignmentEntity,
        comment: String?,
        currentUserId: Long
    ) {
        request.status =
            MetricApplicabilityRequestStatus.REJECTED

        request.decisionBy = currentUserId
        request.updatedAt = LocalDateTime.now()

        assignment.applicabilityStatus =
            MetricApplicabilityStatus.ACTIVE

        /*
         * isVisibleInOffice остаётся true.
         */

        saveHistory(
            request = request,
            action = MetricApplicabilityAction.REJECTED,
            currentUserId = currentUserId,
            comment = comment
        )
    }

    /**
     * Отменяет принятое решение:
     *
     * APPROVED/NOT_APPLICABLE или REJECTED/ACTIVE
     * -> PENDING/PENDING.
     */
    private fun cancelDecision(
        request: MetricApplicabilityRequestEntity,
        assignment: InitiativeMetricAssignmentEntity,
        comment: String?,
        currentUserId: Long
    ) {
        request.status =
            MetricApplicabilityRequestStatus.PENDING

        request.updatedAt = LocalDateTime.now()
        request.effectiveToPeriod = null

        /*
         * decisionBy по требованиям не очищается.
         */

        assignment.applicabilityStatus =
            MetricApplicabilityStatus.PENDING

        saveHistory(
            request = request,
            action = MetricApplicabilityAction.CANCEL_DECISION,
            currentUserId = currentUserId,
            comment = comment
        )
    }

    /**
     * Добавляет новое действие в историю заявки.
     */
    private fun saveHistory(
        request: MetricApplicabilityRequestEntity,
        action: MetricApplicabilityAction,
        currentUserId: Long,
        comment: String?
    ) {
        metricApplicabilityHistoryRepository.save(
            MetricApplicabilityHistoryEntity(
                metricApplicabilityRequest = request,
                action = action,
                createdBy = currentUserId.toString(),
                comment = comment,
                createdAt = LocalDate.now()
            )
        )
    }

    /**
     * Проверяет принадлежность заявки
     * указанному assignment.
     */
    private fun validateRequestBelongsToAssignment(
        request: MetricApplicabilityRequestEntity,
        assignmentId: Long
    ) {
        if (
            request.initiativeMetricAssignment.id !=
            assignmentId
        ) {
            throw AiBadRequestException(
                errorCode = REQUEST_NOT_BELONG_TO_ASSIGNMENT,
                message = MessageFormat.format(
                    messageProvider[
                        REQUEST_NOT_BELONG_TO_ASSIGNMENT
                    ],
                    request.id
                )
            )
        }
    }

    /**
     * Проверяет допустимость действия
     * в текущем состоянии заявки.
     */
    private fun validateAction(
        request: MetricApplicabilityRequestEntity,
        assignment: InitiativeMetricAssignmentEntity,
        action: MetricApplicabilityRequestAction
    ) {
        val availableActions =
            metricApplicabilityActionResolver
                .getAvailableActions(
                    requestStatus = request.status,
                    applicabilityStatus =
                        assignment.applicabilityStatus
                )

        /*
         * Скрытая после RESTORE заявка
         * также недоступна для действий Офиса.
         */
        if (
            !request.isVisibleInOffice ||
            action !in availableActions
        ) {
            throw AiBadRequestException(
                errorCode = ACTION_NOT_AVAILABLE,
                message = MessageFormat.format(
                    messageProvider[ACTION_NOT_AVAILABLE],
                    action
                )
            )
        }
    }

    /**
     * Проверяет существование метрики.
     */
    private fun validateMetricExists(
        metricId: UUID
    ) {
        metricsDirectoryRepository
            .findByIdOrNull(metricId)
            ?: throw AiNotFoundException(
                errorCode = INITIATIVE_METRIC_NOT_FOUND,
                message = MessageFormat.format(
                    messageProvider[
                        INITIATIVE_METRIC_NOT_FOUND
                    ],
                    metricId
                )
            )
    }

    /**
     * Регистрирует отправку email
     * после успешного commit транзакции.
     */
    private fun registerNotificationAfterCommit(
        recipientUserId: Long,
        action: MetricApplicabilityRequestAction,
        metricName: String,
        initiativeName: String
    ) {
        TransactionSynchronizationManager
            .registerSynchronization(
                object : TransactionSynchronization {

                    override fun afterCommit() {
                        runCatching {
                            sendNotification(
                                recipientUserId =
                                    recipientUserId,
                                action = action,
                                metricName = metricName,
                                initiativeName =
                                    initiativeName
                            )
                        }.onFailure { exception ->
                            log.error(
                                "Ошибка отправки уведомления " +
                                    "по заявке на неприменимость " +
                                    "метрики: userId={}, action={}",
                                recipientUserId,
                                action,
                                exception
                            )
                        }
                    }
                }
            )
    }

    /**
     * Получает email автора заявки
     * и отправляет уведомление.
     */
    private fun sendNotification(
        recipientUserId: Long,
        action: MetricApplicabilityRequestAction,
        metricName: String,
        initiativeName: String
    ) {
        val user =
            userAccountService
                .getUsersByIds(
                    setOf(recipientUserId)
                )
                .get(recipientUserId)

        if (user == null) {
            log.warn(
                "Не найден пользователь для отправки " +
                    "уведомления, userId={}",
                recipientUserId
            )
            return
        }

        val email = user.email?.trim()

        if (email.isNullOrEmpty()) {
            log.warn(
                "У пользователя отсутствует email, " +
                    "userId={}",
                recipientUserId
            )
            return
        }

        emailHandler.fillAndSend(
            getEmailTemplate(action),
            listOf(email),
            mutableMapOf(
                METRIC_NAME to metricName,
                INITIATIVE_NAME to initiativeName,
                LINK to
                    emailProperties
                        .emailLinkProperties
                        .linkToPortalShort
            )
        )
    }

    /**
     * Возвращает шаблон письма
     * для выполненного действия.
     */
    private fun getEmailTemplate(
        action: MetricApplicabilityRequestAction
    ): String =
        when (action) {

            MetricApplicabilityRequestAction.APPROVE ->
                UNLINK_METRIC_APPROVE_NOTIFICATION_TEMPLATE

            MetricApplicabilityRequestAction.REJECT ->
                UNLINK_METRIC_REJECT_NOTIFICATION_TEMPLATE

            MetricApplicabilityRequestAction.CANCEL_DECISION ->
                UNLINK_METRIC_CANCEL_DECISION_NOTIFICATION_TEMPLATE
        }
}
/**
 * Выполняет действие Офиса над заявкой
 * на неприменимость метрики.
 *
 * Делегирует изменение состояния заявки
 * в MetricApplicabilityRequestUpdater.
 */
fun updateRequest(
    initiativeId: Long,
    metricId: UUID,
    agentType: String,
    requestId: Long,
    request: UpdateMetricApplicabilityRequest,
): UpdateMetricApplicabilityResponse =
    metricApplicabilityRequestUpdater.updateRequest(
        initiativeId = initiativeId,
        metricId = metricId,
        agentType = agentType,
        requestId = requestId,
        request = request,
    )


/**
 * Выполняет решение Офиса AI-трансформации
 * по заявке на неприменимость метрики.
 *
 * APPROVE:
 * PENDING/PENDING -> APPROVED/NOT_APPLICABLE.
 *
 * REJECT:
 * PENDING/PENDING -> REJECTED/ACTIVE.
 *
 * CANCEL_DECISION:
 * APPROVED/NOT_APPLICABLE или REJECTED/ACTIVE
 * -> PENDING/PENDING.
 */
@PatchMapping(
    "/initiatives/{initiativeId}/metrics/{metricId}/" +
        "{agentType}/applicability-requests/{requestId}"
)
@PreAuthorize("hasAnyAuthority('TRANSFORMATION_OFFICE')")
@Operation(summary = "Изменение решения по заявке на отвязку метрики")
@ExceptionApiResponses
open fun updateRequest(
    @PathVariable("initiativeId")
    @Schema(
        description = "Идентификатор инициативы",
        format = "int64",
    )
    initiativeId: Long,

    @PathVariable("metricId")
    @Schema(
        description = "Идентификатор метрики",
        format = "uuid",
    )
    metricId: UUID,

    @PathVariable("agentType")
    @Schema(
        description = "Тип агента",
        example = "autonomous",
    )
    agentType: String,

    @PathVariable("requestId")
    @Schema(
        description = "Идентификатор заявки",
        format = "int64",
    )
    requestId: Long,

    @RequestBody
    @Valid
    @ValidUpdateUnlinkMetricRequest
    request: UpdateMetricApplicabilityRequest,
): UpdateMetricApplicabilityResponse {
    return metricApplicabilityRequestService.updateRequest(
        initiativeId = initiativeId,
        metricId = metricId,
        agentType = agentType,
        requestId = requestId,
        request = request,
    )
}

```
