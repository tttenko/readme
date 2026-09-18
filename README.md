```java
/**
 * Ответ ручного восстановления применимости метрики.
 */
data class MetricApplicabilityRestoreResponse(

    @field:Schema(
        description = "Идентификатор заявки"
    )
    val requestId: Long,

    @field:Schema(
        description = "Статус заявки после RESTORE"
    )
    val requestStatus: MetricApplicabilityRequestStatus,

    @field:Schema(
        description = "Статус применимости метрики после RESTORE"
    )
    val applicabilityStatus: MetricApplicabilityStatus,

    @field:Schema(
        description = "Доступные действия после RESTORE"
    )
    val availableActions: List<MetricApplicabilityRequestAction>
)

/**
 * Возвращает последнюю заявку assignment.
 *
 * createdAt хранится как DATE, поэтому id используется
 * как дополнительная стабильная сортировка.
 */
fun findFirstByInitiativeMetricAssignmentIdOrderByCreatedAtDescIdDesc(
    initiativeMetricAssignmentId: Long
): MetricApplicabilityRequestEntity?

@Service
class MetricApplicabilityRequestRestorer(
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
     * Вручную восстанавливает применимость метрики.
     *
     * APPROVED / NOT_APPLICABLE
     * ->
     * APPROVED / ACTIVE.
     *
     * После RESTORE заявка исключается из очереди Офиса.
     */
    @Transactional
    fun restore(
        initiativeId: Long,
        metricId: UUID,
        agentType: String,
        requestId: Long
    ): MetricApplicabilityRestoreResponse {

        val currentUserId =
            userInfoProvider.currentUser().id

        /*
         * Проверяем существование связки
         * initiative + agentType.
         */
        val initiativeMetricType =
            initiativeMetricTypeRepository
                .findByAiAgentIdAndAgentType(
                    initiativeId = initiativeId,
                    agentType = agentType
                )
                ?: throw AiNotFoundException(
                    errorCode =
                        INITIATIVE_AGENT_TYPE_NOT_FOUND,
                    message = MessageFormat.format(
                        messageProvider[
                            INITIATIVE_AGENT_TYPE_NOT_FOUND
                        ],
                        initiativeId,
                        agentType
                    )
                )

        val initiativeMetricTypeId =
            initiativeMetricType.id
                ?: error(
                    "У сохранённого типа агента инициативы отсутствует id"
                )

        /*
         * RESTORE разрешён только для существующей
         * и активной справочной метрики.
         */
        val metric =
            metricsDirectoryRepository
                .findByIdOrNull(metricId)
                ?: throw AiNotFoundException(
                    errorCode =
                        INITIATIVE_METRIC_NOT_FOUND,
                    message = MessageFormat.format(
                        messageProvider[
                            INITIATIVE_METRIC_NOT_FOUND
                        ],
                        metricId
                    )
                )

        if (metric.active != true) {
            throw AiBadRequestException(
                errorCode =
                    METRIC_DIRECTORY_INACTIVE,
                message = MessageFormat.format(
                    messageProvider[
                        METRIC_DIRECTORY_INACTIVE
                    ],
                    metricId
                )
            )
        }

        /*
         * Получаем assignment именно для указанной
         * метрики и agentType.
         */
        val assignment =
            initiativeMetricAssignmentRepository
                .findByInitiativeMetricTypeIdAndMetricId(
                    initiativeMetricTypeId =
                        initiativeMetricTypeId,
                    metricId = metricId
                )
                ?: throw AiNotFoundException(
                    errorCode =
                        INITIATIVE_METRIC_ASSIGNMENT_NOT_FOUND,
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
         * Блокируем request до конца транзакции.
         */
        val applicabilityRequest =
            metricApplicabilityRequestRepository
                .findByIdForUpdate(requestId)
                ?: throw AiNotFoundException(
                    errorCode =
                        METRIC_APPLICABILITY_REQUEST_NOT_FOUND,
                    message = MessageFormat.format(
                        messageProvider[
                            METRIC_APPLICABILITY_REQUEST_NOT_FOUND
                        ],
                        requestId
                    )
                )

        /*
         * requestId не должен позволять изменить
         * заявку другого assignment.
         */
        validateRequestBelongsToAssignment(
            request = applicabilityRequest,
            assignmentId = assignmentId
        )

        /*
         * RESTORE можно выполнять только
         * для актуальной заявки assignment.
         */
        validateCurrentRequest(
            request = applicabilityRequest,
            assignmentId = assignmentId
        )

        /*
         * Ожидаем только:
         *
         * request = APPROVED
         * assignment = NOT_APPLICABLE.
         */
        validateRestoreAvailable(
            request = applicabilityRequest,
            assignment = assignment
        )

        /*
         * Сам request остаётся APPROVED.
         *
         * Возвращаем только применимость метрики.
         */
        assignment.applicabilityStatus =
            MetricApplicabilityStatus.ACTIVE

        applicabilityRequest.isVisibleInOffice =
            false

        applicabilityRequest.updatedAt =
            LocalDateTime.now()

        /*
         * Метрика снова применима начиная
         * с текущего отчётного периода.
         */
        applicabilityRequest.effectiveToPeriod =
            LocalDate.now().withDayOfMonth(1)

        /*
         * Добавляем immutable history.
         */
        metricApplicabilityHistoryRepository.save(
            MetricApplicabilityHistoryEntity(
                metricApplicabilityRequest =
                    applicabilityRequest,
                action =
                    MetricApplicabilityAction.RESTORED,
                createdBy =
                    currentUserId.toString(),
                comment = null,
                createdAt = LocalDate.now()
            )
        )

        /*
         * Получателем RESTORE является сотрудник Офиса,
         * который принимал решение по заявке.
         */
        val decisionBy =
            applicabilityRequest.decisionBy

        if (decisionBy != null) {

            /*
             * Все данные Entity получаем до commit.
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
                recipientUserId = decisionBy,
                metricName = metricName,
                initiativeName = initiativeName,
                initiativeId = initiativeId
            )

        } else {
            /*
             * Отсутствие пользователя для email
             * не должно откатывать RESTORE.
             */
            log.warn(
                "Не указан decisionBy для RESTORE, requestId={}",
                requestId
            )
        }

        return MetricApplicabilityRestoreResponse(
            requestId =
                applicabilityRequest.id
                    ?: error(
                        "У сохранённой заявки отсутствует id"
                    ),
            requestStatus =
                applicabilityRequest.status,
            applicabilityStatus =
                assignment.applicabilityStatus,
            availableActions =
                emptyList()
        )
    }

    /**
     * Проверяет принадлежность request
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
                errorCode =
                    REQUEST_NOT_BELONG_TO_ASSIGNMENT,
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
     * Проверяет, что request является
     * актуальной заявкой assignment.
     */
    private fun validateCurrentRequest(
        request: MetricApplicabilityRequestEntity,
        assignmentId: Long
    ) {
        val currentRequest =
            metricApplicabilityRequestRepository
                .findFirstByInitiativeMetricAssignmentIdOrderByCreatedAtDescIdDesc(
                    assignmentId
                )

        if (
            currentRequest?.id !=
            request.id
        ) {
            throw AiBadRequestException(
                errorCode =
                    ACTION_NOT_AVAILABLE,
                message = MessageFormat.format(
                    messageProvider[
                        ACTION_NOT_AVAILABLE
                    ],
                    "RESTORE"
                )
            )
        }
    }

    /**
     * Проверяет допустимость RESTORE.
     *
     * Разрешено только:
     * APPROVED / NOT_APPLICABLE.
     */
    private fun validateRestoreAvailable(
        request: MetricApplicabilityRequestEntity,
        assignment: InitiativeMetricAssignmentEntity
    ) {
        if (
            request.status !=
            MetricApplicabilityRequestStatus.APPROVED ||
            assignment.applicabilityStatus !=
            MetricApplicabilityStatus.NOT_APPLICABLE
        ) {
            throw AiBadRequestException(
                errorCode =
                    ACTION_NOT_AVAILABLE,
                message = MessageFormat.format(
                    messageProvider[
                        ACTION_NOT_AVAILABLE
                    ],
                    "RESTORE"
                )
            )
        }
    }

    /**
     * Регистрирует отправку письма
     * после успешного commit.
     */
    private fun registerNotificationAfterCommit(
        recipientUserId: Long,
        metricName: String,
        initiativeName: String,
        initiativeId: Long
    ) {
        TransactionSynchronizationManager
            .registerSynchronization(
                object : TransactionSynchronization {

                    override fun afterCommit() {
                        runCatching {
                            sendNotification(
                                recipientUserId =
                                    recipientUserId,
                                metricName =
                                    metricName,
                                initiativeName =
                                    initiativeName,
                                initiativeId =
                                    initiativeId
                            )
                        }.onFailure { exception ->
                            log.error(
                                "Ошибка отправки уведомления " +
                                    "после RESTORE: userId={}",
                                recipientUserId,
                                exception
                            )
                        }
                    }
                }
            )
    }

    /**
     * Получает email сотрудника Офиса
     * и отправляет уведомление о RESTORE.
     */
    private fun sendNotification(
        recipientUserId: Long,
        metricName: String,
        initiativeName: String,
        initiativeId: Long
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

        val email =
            user.email?.trim()

        if (email.isNullOrEmpty()) {
            log.warn(
                "У пользователя отсутствует email, " +
                    "userId={}",
                recipientUserId
            )
            return
        }

        /*
         * По текущей аналитике RESTORE использует
         * шаблон Reject.
         *
         * Модель письма формируется так же,
         * как в POST и Office PATCH.
         */
        emailHandler.fillAndSend(
            UNLINK_METRIC_REJECT_NOTIFICATION_TEMPLATE,
            listOf(email),
            mutableMapOf(
                METRIC_NAME to
                    metricName,

                INITIATIVE_NAME to
                    initiativeName,

                SUPPORT_EMAIL to
                    emailProperties
                        .emailLinkProperties
                        .pultSupportBox,

                LINK to
                    PultLinksHelper
                        .buildLinkToInitiative(
                            emailProperties
                                .emailLinkProperties
                                .linkToPortalShort,
                            initiativeId
                        )
            )
        )
    }
}

/**
 * Вручную возвращает применимость
 * ранее согласованной метрики.
 *
 * request остаётся APPROVED,
 * applicabilityStatus становится ACTIVE.
 */
@PatchMapping(
    "/initiatives/{initiativeId}/metrics/{metricId}/" +
        "{agentType}/applicability-requests/{requestId}/restore"
)
@PreAuthorize(
    "hasAnyAuthority('PROJECT_OFFICE', 'CMS_ADMIN')"
)
open fun restoreMetricApplicabilityRequest(
    @PathVariable
    initiativeId: Long,

    @PathVariable
    metricId: UUID,

    @PathVariable
    agentType: String,

    @PathVariable
    requestId: Long
): MetricApplicabilityRestoreResponse =
    metricApplicabilityRequestService
        .restoreMetricApplicabilityRequest(
            initiativeId = initiativeId,
            metricId = metricId,
            agentType = agentType,
            requestId = requestId
        )


/**
 * Выполняет ручное восстановление
 * применимости метрики.
 */
fun restoreMetricApplicabilityRequest(
    initiativeId: Long,
    metricId: UUID,
    agentType: String,
    requestId: Long
): MetricApplicabilityRestoreResponse =
    metricApplicabilityRequestRestorer.restore(
        initiativeId = initiativeId,
        metricId = metricId,
        agentType = agentType,
        requestId = requestId
    )
```
