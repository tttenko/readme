```java
/**
 * Запрос на изменение статуса заявки на неприменимость метрики.
 */
data class UpdateMetricApplicabilityRequest(
    val action: MetricApplicabilityRequestAction,
    val comment: String? = null,
)

/**
 * Результат изменения заявки.
 */
data class UpdateMetricApplicabilityResponse(
    val requestId: Long,
    val requestStatus: MetricApplicabilityRequestStatus,
    val applicabilityStatus: MetricApplicabilityStatus,
    val availableActions: List<MetricApplicabilityRequestAction>,
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



const val METRIC_APPLICABILITY_REQUEST_NOT_FOUND =
        "metric.applicability.request.not.found"


metric.applicability.request.not.found=Заявка на неприменимость метрики с идентификатором {0} не найдена



/**
 * Получает конкретную заявку указанного assignment
 * с блокировкой на изменение.
 *
 * Это не позволяет двум сотрудникам Офиса
 * одновременно принять разные решения по одной заявке.
 */
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query(
    """
        select request
        from MetricApplicabilityRequestEntity request
        where request.id = :requestId
          and request.initiativeMetricAssignment.id = :assignmentId
    """
)
fun findByIdAndAssignmentIdForUpdate(
    @Param("requestId")
    requestId: Long,

    @Param("assignmentId")
    assignmentId: Long,
): MetricApplicabilityRequestEntity?


import org.springframework.data.repository.findByIdOrNull
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import org.springframework.transaction.support.TransactionSynchronization
import org.springframework.transaction.support.TransactionSynchronizationManager
import ru.sber.prm.auth.UserInfoProvider
import ru.sber.prm.config.properties.EmailProperties
import ru.sber.prm.email.EmailHandler
import ru.sber.prm.exception.AiBadRequestException
import ru.sber.prm.exception.AiNotFoundException
import ru.sber.prm.logger.logger
import ru.sber.prm.messageprovider.MessageProvider
import ru.sber.prm.utils.Metadata.Email.LINK
import ru.sber.prm.utils.Metadata.Email.SUPPORT_EMAIL
import ru.sber.prm.utils.Metadata.Email.UNLINK_METRIC_APPROVE_NOTIFICATION_TEMPLATE
import ru.sber.prm.utils.Metadata.Email.UNLINK_METRIC_CANCEL_DECISION_NOTIFICATION_TEMPLATE
import ru.sber.prm.utils.Metadata.Email.UNLINK_METRIC_REJECT_NOTIFICATION_TEMPLATE
import ru.sber.prm.utils.Metadata.ErrorMessages.ACTION_NOT_AVAILABLE
import ru.sber.prm.utils.Metadata.ErrorMessages.INITIATIVE_METRIC_AGENT_TYPE_NOT_FOUND
import ru.sber.prm.utils.Metadata.ErrorMessages.METRIC_APPLICABILITY_REQUEST_NOT_FOUND
import ru.sber.prm.utils.Metadata.ErrorMessages.METRIC_NOT_FOUND
import java.text.MessageFormat
import java.time.LocalDateTime
import java.util.UUID

/**
 * Выполняет решения Офиса AI-трансформации
 * по заявкам на неприменимость метрик.
 *
 * Поддерживаемые действия:
 * APPROVE, REJECT, CANCEL_DECISION.
 *
 * Изменение request, assignment и добавление history
 * выполняются одной транзакцией.
 *
 * Email отправляется только после успешного commit.
 */
@Service
class MetricApplicabilityRequestUpdater(
    private val messageProvider: MessageProvider,
    private val initiativeMetricTypeRepository: InitiativeMetricTypeRepository,
    private val initiativeMetricAssignmentRepository: InitiativeMetricAssignmentRepository,
    private val metricsDirectoryRepository: MetricsDirectoryRepository,
    private val metricApplicabilityRequestRepository: MetricApplicabilityRequestRepository,
    private val metricApplicabilityHistoryRepository: MetricApplicabilityHistoryRepository,
    private val metricApplicabilityActionResolver: MetricApplicabilityActionResolver,
    private val userAccountService: UserAccountService,
    private val emailHandler: EmailHandler,
    private val emailProperties: EmailProperties,
    private val userInfoProvider: UserInfoProvider,
) {

    companion object {
        private val log by logger()
    }

    /**
     * Выполняет действие над заявкой
     * и возвращает фактическое состояние после изменения.
     */
    @Transactional
    @OperationDetails("Process 'update metric applicability request'")
    fun updateRequest(
        initiativeId: Long,
        metricId: UUID,
        agentType: String,
        request: UpdateMetricApplicabilityRequest,
        requestId: Long,
    ): UpdateMetricApplicabilityResponse {

        val userId = userInfoProvider.currentUser().id

        val initiativeMetricType =
            validateInitiativeAgentType(
                initiativeId = initiativeId,
                agentType = agentType.trim(),
            )

        /*
         * Метрика из path должна существовать.
         *
         * Одновременно получаем её name,
         * который понадобится для email.
         */
        val metric =
            metricsDirectoryRepository.findByIdOrNull(metricId)
                ?: throw AiNotFoundException(
                    errorCode = METRIC_NOT_FOUND,
                    message = MessageFormat.format(
                        messageProvider[METRIC_NOT_FOUND],
                        metricId,
                    ),
                )

        /*
         * Assignment определяет конкретную связку:
         * initiative + agentType + metric.
         */
        val assignment =
            initiativeMetricAssignmentRepository
                .findByInitiativeMetricTypeIdAndMetricId(
                    initiativeMetricTypeId = initiativeMetricType.id,
                    metricId = metricId,
                )
                ?: throwRequestNotFound(requestId)

        /*
         * Request получаем именно внутри найденного assignment.
         * Одновременно ставится pessimistic write lock.
         */
        val requestEntity =
            metricApplicabilityRequestRepository
                .findByIdAndAssignmentIdForUpdate(
                    requestId = requestId,
                    assignmentId = assignment.id,
                )
                ?: throwRequestNotFound(requestId)

        validateActionAvailable(
            request = requestEntity,
            assignment = assignment,
            action = request.action,
        )

        val comment =
            request.comment
                ?.trim()
                ?.takeIf { it.isNotEmpty() }

        val historyAction =
            when (request.action) {

                MetricApplicabilityRequestAction.APPROVE -> {
                    approve(
                        request = requestEntity,
                        assignment = assignment,
                        userId = userId,
                    )

                    MetricApplicabilityAction.APPROVED
                }

                MetricApplicabilityRequestAction.REJECT -> {
                    reject(
                        request = requestEntity,
                        assignment = assignment,
                        userId = userId,
                    )

                    MetricApplicabilityAction.REJECTED
                }

                MetricApplicabilityRequestAction.CANCEL_DECISION -> {
                    cancelDecision(
                        request = requestEntity,
                        assignment = assignment,
                    )

                    MetricApplicabilityAction.CANCEL_DECISION
                }
            }

        /*
         * UPDATE request и assignment.
         *
         * Объекты и так managed внутри транзакции,
         * но save оставляем в стиле текущего проекта явно.
         */
        initiativeMetricAssignmentRepository.save(assignment)
        metricApplicabilityRequestRepository.save(requestEntity)

        /*
         * Каждое действие создаёт новую строку history.
         * Старые записи не меняются.
         */
        metricApplicabilityHistoryRepository.save(
            MetricApplicabilityHistoryEntity(
                metricApplicabilityRequest = requestEntity,
                action = historyAction,
                createdBy = userId.toString(),
                comment = comment,
            )
        )

        /*
         * Значения для письма собираем до окончания транзакции,
         * чтобы afterCommit не зависел от lazy JPA relations.
         */
        val agentName =
            initiativeMetricType.aiAgent.agentName.orEmpty()

        val metricName =
            metric.name.orEmpty()

        sendNotificationAfterCommit(
            requestId = requestEntity.id,
            recipientUserId = requestEntity.createdBy,
            action = request.action,
            initiativeId = initiativeId,
            agentName = agentName,
            metricName = metricName,
        )

        val applicabilityStatus =
            assignment.applicabilityStatus

        return UpdateMetricApplicabilityResponse(
            requestId = requestEntity.id,
            requestStatus = requestEntity.status,
            applicabilityStatus = applicabilityStatus,
            availableActions =
                metricApplicabilityActionResolver.getAvailableActions(
                    requestStatus = requestEntity.status,
                    applicabilityStatus = applicabilityStatus,
                ),
        )
    }

    /**
     * APPROVE:
     * PENDING / PENDING
     * ->
     * APPROVED / NOT_APPLICABLE.
     */
    private fun approve(
        request: MetricApplicabilityRequestEntity,
        assignment: InitiativeMetricAssignmentEntity,
        userId: Long,
    ) {
        request.status =
            MetricApplicabilityRequestStatus.APPROVED

        request.decisionBy = userId
        request.updatedAt = LocalDateTime.now()

        /*
         * Если resumePeriod не указан,
         * effectiveToPeriod остаётся null.
         */
        request.effectiveToPeriod =
            request.resumePeriod

        assignment.applicabilityStatus =
            MetricApplicabilityStatus.NOT_APPLICABLE
    }

    /**
     * REJECT:
     * PENDING / PENDING
     * ->
     * REJECTED / ACTIVE.
     */
    private fun reject(
        request: MetricApplicabilityRequestEntity,
        assignment: InitiativeMetricAssignmentEntity,
        userId: Long,
    ) {
        request.status =
            MetricApplicabilityRequestStatus.REJECTED

        request.decisionBy = userId
        request.updatedAt = LocalDateTime.now()

        /*
         * isVisibleInOffice остаётся true.
         */
        assignment.applicabilityStatus =
            MetricApplicabilityStatus.ACTIVE
    }

    /**
     * CANCEL_DECISION:
     *
     * APPROVED / NOT_APPLICABLE
     * или
     * REJECTED / ACTIVE
     *
     * ->
     *
     * PENDING / PENDING.
     */
    private fun cancelDecision(
        request: MetricApplicabilityRequestEntity,
        assignment: InitiativeMetricAssignmentEntity,
    ) {
        request.status =
            MetricApplicabilityRequestStatus.PENDING

        request.updatedAt = LocalDateTime.now()

        /*
         * Предыдущее ограничение по сроку
         * больше не действует.
         */
        request.effectiveToPeriod = null

        /*
         * decisionBy специально не очищаем.
         *
         * Если заявку позже рассмотрит другой сотрудник,
         * APPROVE/REJECT перезапишет decisionBy.
         */

        assignment.applicabilityStatus =
            MetricApplicabilityStatus.PENDING
    }

    /**
     * Проверяет доступность действия
     * для текущего состояния request + assignment.
     *
     * Используется тот же resolver,
     * что и в GET очереди.
     */
    private fun validateActionAvailable(
        request: MetricApplicabilityRequestEntity,
        assignment: InitiativeMetricAssignmentEntity,
        action: MetricApplicabilityRequestAction,
    ) {
        val availableActions =
            metricApplicabilityActionResolver.getAvailableActions(
                requestStatus = request.status,
                applicabilityStatus = assignment.applicabilityStatus,
            )

        if (
            !request.isVisibleInOffice ||
            action !in availableActions
        ) {
            throw AiBadRequestException(
                errorCode = ACTION_NOT_AVAILABLE,
                message = MessageFormat.format(
                    messageProvider[ACTION_NOT_AVAILABLE],
                    "${request.status}/${assignment.applicabilityStatus}",
                ),
            )
        }
    }

    /**
     * Проверяет наличие типа агента
     * у указанной инициативы.
     */
    private fun validateInitiativeAgentType(
        initiativeId: Long,
        agentType: String,
    ): InitiativeMetricTypeEntity =
        initiativeMetricTypeRepository
            .findByAiAgentIdAndAgentType(
                initiativeId,
                agentType,
            )
            ?: throw AiNotFoundException(
                errorCode = INITIATIVE_METRIC_AGENT_TYPE_NOT_FOUND,
                message = MessageFormat.format(
                    messageProvider[
                        INITIATIVE_METRIC_AGENT_TYPE_NOT_FOUND
                    ],
                    initiativeId,
                    agentType,
                ),
            )

    /**
     * Формирует стандартную ошибку,
     * когда request не найден в указанном assignment.
     */
    private fun throwRequestNotFound(
        requestId: Long,
    ): Nothing {
        throw AiNotFoundException(
            errorCode = METRIC_APPLICABILITY_REQUEST_NOT_FOUND,
            message = MessageFormat.format(
                messageProvider[
                    METRIC_APPLICABILITY_REQUEST_NOT_FOUND
                ],
                requestId,
            ),
        )
    }

    /**
     * Регистрирует отправку email
     * только после успешного commit.
     */
    private fun sendNotificationAfterCommit(
        requestId: Long,
        recipientUserId: Long,
        action: MetricApplicabilityRequestAction,
        initiativeId: Long,
        agentName: String,
        metricName: String,
    ) {
        TransactionSynchronizationManager.registerSynchronization(
            object : TransactionSynchronization {

                override fun afterCommit() {
                    runCatching {
                        sendNotification(
                            recipientUserId = recipientUserId,
                            action = action,
                            initiativeId = initiativeId,
                            agentName = agentName,
                            metricName = metricName,
                        )
                    }.onFailure { exception ->
                        log.error(
                            "Failed to send notification " +
                                "for metric applicability request " +
                                "requestId=$requestId, action=$action",
                            exception,
                        )
                    }
                }
            }
        )
    }

    /**
     * Получает email автора заявки
     * и отправляет письмо по соответствующему шаблону.
     */
    private fun sendNotification(
        recipientUserId: Long,
        action: MetricApplicabilityRequestAction,
        initiativeId: Long,
        agentName: String,
        metricName: String,
    ) {
        val user =
            userAccountService
                .getUsersByIds(setOf(recipientUserId))
                [recipientUserId]

        if (user == null) {
            log.warn(
                "User not found for metric applicability notification, " +
                    "userId=$recipientUserId",
            )
            return
        }

        val email =
            user.email
                ?.trim()
                ?.takeIf { it.isNotEmpty() }

        if (email == null) {
            log.warn(
                "Email not found for metric applicability notification, " +
                    "userId=$recipientUserId",
            )
            return
        }

        emailHandler.fillAndSend(
            getEmailTemplate(action),
            listOf(email),
            mutableMapOf(
                Pair("agentName", agentName),
                Pair("metric", metricName),
                Pair(
                    SUPPORT_EMAIL,
                    emailProperties
                        .emailLinkProperties
                        .pultSupportBox,
                ),
                Pair(
                    LINK,
                    PultLinksHelper.buildLinkToInitiative(
                        emailProperties
                            .emailLinkProperties
                            .linkToPortalShort,
                        initiativeId,
                    ),
                ),
            ),
        )
    }

    /**
     * Возвращает шаблон письма
     * для конкретного решения Офиса.
     */
    private fun getEmailTemplate(
        action: MetricApplicabilityRequestAction,
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
 * Выполняет решение Офиса
 * по заявке на неприменимость метрики.
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
