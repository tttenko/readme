```java
/**
 * Сервис чтения очереди заявок
 * на неприменимость метрик.
 *
 * Отвечает за:
 * - фильтрацию;
 * - пагинацию;
 * - сортировку;
 * - получение ФИО авторов;
 * - расчёт pendingCount;
 * - формирование DTO ответа.
 */
@Service
class MetricApplicabilityRequestQueryService(
    private val metricApplicabilityRequestRepository:
        MetricApplicabilityRequestRepository,

    private val metricApplicabilityActionResolver:
        MetricApplicabilityActionResolver,

    private val userAccountService:
        UserAccountService
) {

    /**
     * Возвращает страницу заявок,
     * отображаемых в очереди Офиса.
     */
    @Transactional(readOnly = true)
    fun getMetricApplicabilityRequests(
        status: MetricApplicabilityRequestStatus?,
        page: Int,
        size: Int,
        search: String?
    ): MetricApplicabilityRequestsResponse {

        val pageable =
            PageRequest.of(
                page,
                size,
                Sort.by(
                    Sort.Order.desc("createdAt"),
                    Sort.Order.desc("id")
                )
            )

        val specification =
            MetricApplicabilityRequestSpecification
                .buildSpecification(
                    status = status,
                    search = search
                )

        val requestsPage =
            metricApplicabilityRequestRepository
                .findAll(
                    specification,
                    pageable
                )

        /*
         * Получаем всех авторов страницы одним batch-запросом,
         * чтобы не обращаться в prm-auth для каждой заявки.
         */
        val requestedByUserIds =
            requestsPage.content
                .map { it.createdBy }
                .toSet()

        val usersById =
            userAccountService
                .getUsersByIds(
                    requestedByUserIds
                )

        val requests =
            requestsPage.content.map { request ->

                val requestedBy =
                    usersById[
                        request.createdBy
                    ]
                        ?.let(
                            userAccountService::getFullName
                        )
                        ?: request.createdBy.toString()

                request.toResponse(
                    requestedBy = requestedBy
                )
            }

        return MetricApplicabilityRequestsResponse(
            content = requests,
            page = requestsPage.number,
            size = requestsPage.size,
            totalElements =
                requestsPage.totalElements,
            totalPages =
                requestsPage.totalPages,
            pendingCount =
                getRequestsCount(status)
        )
    }

    /**
     * Возвращает количество заявок
     * без учёта текущей страницы.
     *
     * Если status не указан —
     * считаются все заявки, отображаемые Офису.
     */
    private fun getRequestsCount(
        status: MetricApplicabilityRequestStatus?
    ): Long =
        if (status == null) {
            metricApplicabilityRequestRepository
                .countByIsVisibleInOfficeTrue()
        } else {
            metricApplicabilityRequestRepository
                .countByStatusAndIsVisibleInOfficeTrue(
                    status
                )
        }

    /**
     * Преобразует Entity заявки
     * в DTO очереди Офиса.
     */
    private fun MetricApplicabilityRequestEntity.toResponse(
        requestedBy: String
    ): MetricApplicabilityRequestResponse {

        val initiativeMetricType =
            initiativeMetricAssignment
                .initiativeMetricType

        val aiAgent =
            initiativeMetricType.aiAgent

        val metric =
            initiativeMetricAssignment.metric

        return MetricApplicabilityRequestResponse(
            requestId = id,
            initiativeId = aiAgent.id,
            initiativeName = aiAgent.agentName,
            metricId = metric.id,
            metricName = metric.name,
            metricFrequency = metric.frequency,
            agentType =
                initiativeMetricType.agentType,
            requestStatus = status,
            applicabilityStatus =
                initiativeMetricAssignment
                    .applicabilityStatus,
            comment = comment,
            resumePeriod = resumePeriod,
            requestedBy = requestedBy,
            requestedAt = createdAt,
            availableActions =
                metricApplicabilityActionResolver
                    .getAvailableActions(
                        requestStatus = status,
                        applicabilityStatus =
                            initiativeMetricAssignment
                                .applicabilityStatus
                    )
        )
    }
}

/**
     * Возвращает очередь заявок Офиса.
     */
    fun getMetricApplicabilityRequests(
        status: MetricApplicabilityRequestStatus?,
        page: Int,
        size: Int,
        search: String?
    ): MetricApplicabilityRequestsResponse =
        metricApplicabilityRequestQueryService
            .getMetricApplicabilityRequests(
                status = status,
                page = page,
                size = size,
                search = search
            )
```
