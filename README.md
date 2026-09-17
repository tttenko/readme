```java
data class UpdateMetricApplicabilityRequest(

    @field:Schema(
        description = "Действие над заявкой",
        example = "APPROVE"
    )
    val action: MetricApplicabilityRequestAction,

    @field:Schema(
        description = "Комментарий. Обязателен для REJECT и CANCEL_DECISION",
        nullable = true,
        maxLength = 1000
    )
    val comment: String? = null
)

data class MetricApplicabilityRequestActionResponse(

    @field:Schema(
        description = "Идентификатор заявки"
    )
    val requestId: Long,

    @field:Schema(
        description = "Статус заявки"
    )
    val requestStatus: MetricApplicabilityRequestStatus,

    @field:Schema(
        description = "Статус применимости метрики"
    )
    val applicabilityStatus: MetricApplicabilityStatus,

    @field:Schema(
        description = "Количество заявок со статусом PENDING, отображаемых в очереди Офиса"
    )
    val pendingCount: Long,

    @field:Schema(
        description = "Доступные действия для текущего состояния заявки"
    )
    val availableActions: List<MetricApplicabilityRequestAction>
)

/**
 * Проверяет корректность тела запроса
 * при изменении заявки на неприменимость метрики.
 */
@Target(AnnotationTarget.VALUE_PARAMETER)
@Retention(AnnotationRetention.RUNTIME)
@Constraint(validatedBy = [UpdateUnlinkMetricRequestValidator::class])
annotation class ValidUpdateUnlinkMetricRequest(
    val message: String = "VALIDATION_ERROR",
    val groups: Array<KClass<*>> = [],
    val payload: Array<KClass<out Payload>> = [],
)

@Component
class UpdateUnlinkMetricRequestValidator(
    private val messageProvider: MessageProvider,
) : ConstraintValidator<
    ValidUpdateUnlinkMetricRequest,
    UpdateMetricApplicabilityRequest
> {

    private companion object {
        const val COMMENT_MAX_LENGTH = 1000
    }

    override fun isValid(
        request: UpdateMetricApplicabilityRequest,
        context: ConstraintValidatorContext,
    ): Boolean {
        context.disableDefaultConstraintViolation()

        val comment = request.comment?.trim()

        if (comment != null && comment.length > COMMENT_MAX_LENGTH) {
            val message = messageProvider[COMMENT_TOO_LONG]

            context
                .buildConstraintViolationWithTemplate(message)
                .addConstraintViolation()

            return false
        }

        if (
            request.action in setOf(
                MetricApplicabilityRequestAction.REJECT,
                MetricApplicabilityRequestAction.CANCEL_DECISION,
            ) &&
            comment.isNullOrEmpty()
        ) {
            val message = messageProvider[COMMENT_REQUIRED]

            context
                .buildConstraintViolationWithTemplate(message)
                .addConstraintViolation()

            return false
        }

        return true
    }
}

const val UNLINK_METRIC_APPROVE_NOTIFICATION_TEMPLATE =
        "unlinkMetricFromInitiativeRequestApprove"

    const val UNLINK_METRIC_REJECT_NOTIFICATION_TEMPLATE =
        "unlinkMetricFromInitiativeRequestReject"

    const val UNLINK_METRIC_CANCEL_DECISION_NOTIFICATION_TEMPLATE =
        "unlinkMetricFromInitiativeRequestCancelDecision"



const val INITIATIVE_AGENT_TYPE_NOT_FOUND =
    "initiative.agent.type.not.found"

const val COMMENT_REQUIRED = "comment.required"
comment.required=Комментарий обязателен для выбранного действия



/**
     * Получает заявку с блокировкой записи на время транзакции.
     */
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query(
        """
        select r
        from MetricApplicabilityRequestEntity r
        where r.id = :requestId
        """
    )
    fun findByIdForUpdate(
        @Param("requestId") requestId: Long
    ): MetricApplicabilityRequestEntity?

    /**
     * Количество видимых в очереди Офиса заявок,
     * ожидающих принятия решения.
     */
    fun countByStatusAndIsVisibleInOfficeTrue(
        status: MetricApplicabilityRequestStatus
    ): Long


fun findByInitiativeAgentTypeIdAndMetricDirectoryId(
        initiativeAgentTypeId: Long,
        metricDirectoryId: UUID
    ): InitiativeMetricAssignmentEntity?




@Service
class MetricApplicabilityRequestUpdater(
    private val metricApplicabilityRequestRepository:
        MetricApplicabilityRequestRepository,

    private val initiativeMetricAssignmentRepository:
        InitiativeMetricAssignmentRepository,

    private val metricApplicabilityHistoryRepository:
        MetricApplicabilityHistoryRepository,

    private val metricApplicabilityActionResolver:
        MetricApplicabilityActionResolver,

    private val initiativeAgentTypeRepository:
        InitiativeAgentTypeRepository,

    private val metricsDirectoryRepository:
        MetricsDirectoryRepository,

    private val userInfoProvider: UserInfoProvider,

    private val userAccountService: UserAccountService,

    private val emailHandler: EmailHandler,

    private val emailProperties: EmailProperties,

    private val messageProvider: MessageProvider
) {

    private val log by logger

    /**
     * Выполняет действие над заявкой на неприменимость метрики.
     *
     * APPROVE:
     * PENDING/PENDING -> APPROVED/NOT_APPLICABLE
     *
     * REJECT:
     * PENDING/PENDING -> REJECTED/ACTIVE
     *
     * CANCEL_DECISION:
     * APPROVED/NOT_APPLICABLE или REJECTED/ACTIVE
     * -> PENDING/PENDING
     */
    @Transactional
    fun updateRequest(
        initiativeId: Long,
        metricId: UUID,
        agentType: String,
        requestId: Long,
        request: MetricApplicabilityRequestUpdateRequest
    ): MetricApplicabilityRequestUpdateResponse {

        val initiativeAgentType =
            initiativeAgentTypeRepository
                .findByInitiativeIdAndAgentType(
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

        validateMetricExists(metricId)

        val assignment =
            initiativeMetricAssignmentRepository
                .findByInitiativeAgentTypeIdAndMetricDirectoryId(
                    initiativeAgentType.id,
                    metricId
                )
                ?: throw AiNotFoundException(
                    errorCode = INITIATIVE_METRIC_ASSIGNMENT_NOT_FOUND,
                    message = MessageFormat.format(
                        messageProvider[INITIATIVE_METRIC_ASSIGNMENT_NOT_FOUND],
                        initiativeId,
                        metricId,
                        agentType
                    )
                )

        val applicabilityRequest =
            metricApplicabilityRequestRepository
                .findByIdForUpdate(requestId)
                ?: throw AiNotFoundException(
                    errorCode = METRIC_APPLICABILITY_REQUEST_NOT_FOUND,
                    message = MessageFormat.format(
                        messageProvider[METRIC_APPLICABILITY_REQUEST_NOT_FOUND],
                        requestId
                    )
                )

        validateRequestBelongsToAssignment(
            request = applicabilityRequest,
            assignmentId = assignment.id
        )

        validateAction(
            request = applicabilityRequest,
            assignment = assignment,
            action = request.action
        )

        when (request.action) {
            MetricApplicabilityRequestAction.APPROVE ->
                approve(
                    applicabilityRequest = applicabilityRequest,
                    assignment = assignment,
                    comment = request.comment
                )

            MetricApplicabilityRequestAction.REJECT ->
                reject(
                    applicabilityRequest = applicabilityRequest,
                    assignment = assignment,
                    comment = request.comment
                )

            MetricApplicabilityRequestAction.CANCEL_DECISION ->
                cancelDecision(
                    applicabilityRequest = applicabilityRequest,
                    assignment = assignment,
                    comment = request.comment
                )
        }

        val pendingCount =
            metricApplicabilityRequestRepository
                .countByStatusAndIsVisibleInOfficeTrue(
                    MetricApplicabilityRequestStatus.PENDING
                )

        val availableActions =
            metricApplicabilityActionResolver.getAvailableActions(
                requestStatus = applicabilityRequest.status,
                applicabilityStatus = assignment.applicabilityStatus
            )

        registerNotificationAfterCommit(
            recipientUserId = applicabilityRequest.createdBy,
            action = request.action
        )

        return MetricApplicabilityRequestUpdateResponse(
            requestId = applicabilityRequest.id,
            requestStatus = applicabilityRequest.status,
            applicabilityStatus = assignment.applicabilityStatus,
            pendingCount = pendingCount,
            availableActions = availableActions
        )
    }

    /**
     * APPROVE:
     * PENDING/PENDING -> APPROVED/NOT_APPLICABLE.
     */
    private fun approve(
        applicabilityRequest: MetricApplicabilityRequestEntity,
        assignment: InitiativeMetricAssignmentEntity,
        comment: String?
    ) {
        val currentUserId = getCurrentUserId()

        applicabilityRequest.status =
            MetricApplicabilityRequestStatus.APPROVED

        applicabilityRequest.decisionBy = currentUserId
        applicabilityRequest.updatedAt = LocalDateTime.now()

        applicabilityRequest.effectiveToPeriod =
            applicabilityRequest.resumePeriod

        assignment.applicabilityStatus =
            MetricApplicabilityStatus.NOT_APPLICABLE

        metricApplicabilityHistoryRepository.save(
            MetricApplicabilityHistoryEntity(
                request = applicabilityRequest,
                action = MetricApplicabilityAction.APPROVED,
                createdBy = currentUserId,
                comment = comment,
                createdAt = LocalDate.now()
            )
        )
    }

    /**
     * REJECT:
     * PENDING/PENDING -> REJECTED/ACTIVE.
     */
    private fun reject(
        applicabilityRequest: MetricApplicabilityRequestEntity,
        assignment: InitiativeMetricAssignmentEntity,
        comment: String?
    ) {
        val currentUserId = getCurrentUserId()

        applicabilityRequest.status =
            MetricApplicabilityRequestStatus.REJECTED

        applicabilityRequest.decisionBy = currentUserId
        applicabilityRequest.updatedAt = LocalDateTime.now()

        assignment.applicabilityStatus =
            MetricApplicabilityStatus.ACTIVE

        /*
         * isVisibleInOffice намеренно не меняем.
         * Согласно спецификации заявка после REJECT
         * остаётся видимой в очереди Офиса.
         */

        metricApplicabilityHistoryRepository.save(
            MetricApplicabilityHistoryEntity(
                request = applicabilityRequest,
                action = MetricApplicabilityAction.REJECTED,
                createdBy = currentUserId,
                comment = comment,
                createdAt = LocalDate.now()
            )
        )
    }

    /**
     * CANCEL_DECISION:
     * APPROVED/NOT_APPLICABLE или REJECTED/ACTIVE
     * -> PENDING/PENDING.
     */
    private fun cancelDecision(
        applicabilityRequest: MetricApplicabilityRequestEntity,
        assignment: InitiativeMetricAssignmentEntity,
        comment: String?
    ) {
        val currentUserId = getCurrentUserId()

        applicabilityRequest.status =
            MetricApplicabilityRequestStatus.PENDING

        applicabilityRequest.updatedAt = LocalDateTime.now()
        applicabilityRequest.effectiveToPeriod = null

        /*
         * decisionBy намеренно не очищаем.
         * Последний принявший решение пользователь
         * должен сохраниться.
         */

        assignment.applicabilityStatus =
            MetricApplicabilityStatus.PENDING

        metricApplicabilityHistoryRepository.save(
            MetricApplicabilityHistoryEntity(
                request = applicabilityRequest,
                action = MetricApplicabilityAction.CANCEL_DECISION,
                createdBy = currentUserId,
                comment = comment,
                createdAt = LocalDate.now()
            )
        )
    }

    /**
     * Проверяет, что заявка относится именно к найденному assignment.
     */
    private fun validateRequestBelongsToAssignment(
        request: MetricApplicabilityRequestEntity,
        assignmentId: Long
    ) {
        if (request.assignment.id != assignmentId) {
            throw AiBadRequestException(
                errorCode = REQUEST_NOT_BELONG_TO_ASSIGNMENT,
                message = MessageFormat.format(
                    messageProvider[REQUEST_NOT_BELONG_TO_ASSIGNMENT],
                    request.id
                )
            )
        }
    }

    /**
     * Проверяет допустимость действия для текущего состояния.
     */
    private fun validateAction(
        request: MetricApplicabilityRequestEntity,
        assignment: InitiativeMetricAssignmentEntity,
        action: MetricApplicabilityRequestAction
    ) {
        val availableActions =
            metricApplicabilityActionResolver.getAvailableActions(
                requestStatus = request.status,
                applicabilityStatus = assignment.applicabilityStatus
            )

        if (action !in availableActions) {
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
    private fun validateMetricExists(metricId: UUID) {
        metricsDirectoryRepository.findByIdOrNull(metricId)
            ?: throw AiNotFoundException(
                errorCode = INITIATIVE_METRIC_NOT_FOUND,
                message = MessageFormat.format(
                    messageProvider[INITIATIVE_METRIC_NOT_FOUND],
                    metricId
                )
            )
    }

    /**
     * Регистрирует отправку уведомления после успешного commit транзакции.
     *
     * Если отправка не удалась, изменение состояния заявки
     * не откатывается.
     */
    private fun registerNotificationAfterCommit(
        recipientUserId: Long,
        action: MetricApplicabilityRequestAction
    ) {
        TransactionSynchronizationManager.registerSynchronization(
            object : TransactionSynchronization {

                override fun afterCommit() {
                    runCatching {
                        sendNotification(
                            recipientUserId = recipientUserId,
                            action = action
                        )
                    }.onFailure { exception ->
                        log.error(
                            "Ошибка отправки уведомления. " +
                                "userId=$recipientUserId, action=$action",
                            exception
                        )
                    }
                }
            }
        )
    }

    /**
     * Получает email пользователя и отправляет уведомление.
     */
    private fun sendNotification(
        recipientUserId: Long,
        action: MetricApplicabilityRequestAction
    ) {
        val user =
            userAccountService
                .getUsersByIds(setOf(recipientUserId))
                [recipientUserId]

        if (user == null) {
            log.warn(
                "Не найден пользователь для отправки уведомления, " +
                    "userId=$recipientUserId"
            )
            return
        }

        val email =
            user.email
                ?.trim()
                ?.takeIf { it.isNotEmpty() }

        if (email == null) {
            log.warn(
                "У пользователя отсутствует email, " +
                    "userId=$recipientUserId"
            )
            return
        }

        emailHandler.fillAndSend(
            getEmailTemplate(action),
            listOf(email),
            mutableMapOf()
        )
    }

    /**
     * Возвращает template уведомления для действия.
     */
    private fun getEmailTemplate(
        action: MetricApplicabilityRequestAction
    ): String =
        when (action) {
            MetricApplicabilityRequestAction.APPROVE ->
                APPROVE_EMAIL_TEMPLATE

            MetricApplicabilityRequestAction.REJECT ->
                REJECT_EMAIL_TEMPLATE

            MetricApplicabilityRequestAction.CANCEL_DECISION ->
                CANCEL_DECISION_EMAIL_TEMPLATE
        }

    /**
     * Возвращает ID текущего пользователя.
     */
    private fun getCurrentUserId(): Long =
        userInfoProvider.user.id
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
