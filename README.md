```java
/**
 * Действия, которые доступны пользователю для заявки на неприменимость метрики.
 */
enum class MetricApplicabilityAction {
    APPROVE,
    REJECT,
    CANCEL_DECISION
}

/**
 * Данные заявки на неприменимость метрики для отображения в очереди Офиса.
 */
data class MetricApplicabilityRequestResponse(
    val requestId: Long,
    val initiativeId: Long,
    val initiativeName: String?,
    val metricId: UUID,
    val metricName: String?,
    val metricFrequency: String?,
    val agentType: String?,
    val requestStatus: MetricApplicabilityRequestStatus,
    val applicabilityStatus: MetricApplicabilityStatus,
    val comment: String,
    val resumePeriod: LocalDate?,
    val requestedBy: String,
    val requestedAt: LocalDate,
    val availableActions: List<MetricApplicabilityAvailableAction>
)

/**
 * Страничный ответ со списком заявок на неприменимость метрик.
 *
 * pendingCount содержит количество видимых заявок выбранного статуса.
 * Если статус не передан, содержит количество всех видимых заявок.
 */
data class MetricApplicabilityRequestsResponse(
    val content: List<MetricApplicabilityRequestResponse>,
    val page: Int,
    val size: Int,
    val totalElements: Long,
    val totalPages: Int,
    val pendingCount: Long
)

/**
 * Определяет доступные действия по текущему состоянию заявки и привязки метрики.
 */
@Component
class MetricApplicabilityActionResolver {

    /**
     * Возвращает список действий, доступных для текущей пары статусов.
     */
    fun getAvailableActions(
        requestStatus: MetricApplicabilityRequestStatus,
        applicabilityStatus: MetricApplicabilityStatus
    ): List<MetricApplicabilityAvailableAction> =
        when {
            requestStatus == MetricApplicabilityRequestStatus.PENDING &&
                applicabilityStatus == MetricApplicabilityStatus.PENDING ->
                listOf(
                    MetricApplicabilityAvailableAction.APPROVE,
                    MetricApplicabilityAvailableAction.REJECT
                )

            requestStatus == MetricApplicabilityRequestStatus.APPROVED &&
                applicabilityStatus == MetricApplicabilityStatus.NOT_APPLICABLE ->
                listOf(MetricApplicabilityAvailableAction.CANCEL_DECISION)

            requestStatus == MetricApplicabilityRequestStatus.REJECTED &&
                applicabilityStatus == MetricApplicabilityStatus.ACTIVE ->
                listOf(MetricApplicabilityAvailableAction.CANCEL_DECISION)

            else -> emptyList()
        }
}

/**
 * Specification для фильтрации заявок на неприменимость метрик.
 */
object MetricApplicabilityRequestSpecification {

    /**
     * Формирует итоговый фильтр очереди Офиса.
     *
     * Всегда возвращаются только заявки с isVisibleInOffice=true.
     * Дополнительно поддерживается фильтрация по статусу и поиск
     * по названию инициативы или метрики.
     */
    fun getSpecification(
        status: MetricApplicabilityRequestStatus?,
        search: String?
    ): Specification<MetricApplicabilityRequestEntity> {
        var specification = isVisibleInOffice()

        if (status != null) {
            specification = specification.and(hasStatus(status))
        }

        if (!search.isNullOrBlank()) {
            specification = specification.and(matchesSearch(search.trim()))
        }

        return specification
    }

    /**
     * Ограничивает выборку заявками, которые должны отображаться Офису.
     */
    private fun isVisibleInOffice(): Specification<MetricApplicabilityRequestEntity> =
        Specification { root, _, criteriaBuilder ->
            criteriaBuilder.isTrue(root.get("isVisibleInOffice"))
        }

    /**
     * Ограничивает выборку заявками с указанным статусом.
     */
    private fun hasStatus(status: MetricApplicabilityRequestStatus): Specification<MetricApplicabilityRequestEntity> =
        Specification { root, _, criteriaBuilder ->
            criteriaBuilder.equal(root.get<MetricApplicabilityRequestStatus>("status"), status)
        }

    /**
     * Выполняет регистронезависимый поиск по названию инициативы и названию метрики.
     */
    private fun matchesSearch(search: String): Specification<MetricApplicabilityRequestEntity> =
        Specification { root, _, criteriaBuilder ->
            val assignment = root.join<MetricApplicabilityRequestEntity, InitiativeMetricAssignmentEntity>(
                "initiativeMetricAssignment",
                JoinType.INNER
            )

            val initiativeMetricType = assignment.join<InitiativeMetricAssignmentEntity, InitiativeMetricTypeEntity>(
                "initiativeMetricType",
                JoinType.INNER
            )

            val initiative = initiativeMetricType.join<InitiativeMetricTypeEntity, AIAgentEntity>(
                "aiAgent",
                JoinType.INNER
            )

            val metric = assignment.join<InitiativeMetricAssignmentEntity, MetricsDirectoryEntity>(
                "metric",
                JoinType.INNER
            )

            val searchPattern = "%${search.lowercase(Locale.ROOT)}%"

            criteriaBuilder.or(
                criteriaBuilder.like(criteriaBuilder.lower(initiative.get<String>("agentName")), searchPattern),
                criteriaBuilder.like(criteriaBuilder.lower(metric.get<String>("name")), searchPattern)
            )
        }
}

@Repository
interface MetricApplicabilityRequestRepository :
    JpaRepository<MetricApplicabilityRequestEntity, Long>,
    JpaSpecificationExecutor<MetricApplicabilityRequestEntity> {

    /**
     * Получает страницу заявок вместе со связями, необходимыми для формирования ответа.
     *
     * EntityGraph позволяет избежать дополнительных запросов к БД
     * для каждой заявки при обращении к инициативе и метрике.
     */
    @EntityGraph(
        attributePaths = [
            "initiativeMetricAssignment",
            "initiativeMetricAssignment.initiativeMetricType",
            "initiativeMetricAssignment.initiativeMetricType.aiAgent",
            "initiativeMetricAssignment.metric"
        ]
    )
    override fun findAll(
        specification: Specification<MetricApplicabilityRequestEntity>?,
        pageable: Pageable
    ): Page<MetricApplicabilityRequestEntity>

    /**
     * Возвращает количество видимых заявок указанного статуса.
     */
    fun countByStatusAndIsVisibleInOfficeTrue(status: MetricApplicabilityRequestStatus): Long

    /**
     * Возвращает общее количество заявок, отображаемых Офису.
     */
    fun countByIsVisibleInOfficeTrue(): Long
}

/**
 * Сервис чтения очереди заявок на неприменимость метрик.
 */
@Service
class MetricApplicabilityRequestService(
    private val metricApplicabilityRequestRepository: MetricApplicabilityRequestRepository,
    private val metricApplicabilityActionResolver: MetricApplicabilityActionResolver,
    private val userInfoProvider: UserInfoProvider
) {

    companion object {
        private const val MAX_PAGE_SIZE = 100
    }

    /**
     * Возвращает страницу заявок на неприменимость метрик.
     *
     * Поддерживает фильтрацию по статусу, поиск по инициативе и метрике,
     * пагинацию и расчёт доступных действий для каждой заявки.
     */
    @Transactional(readOnly = true)
    fun getMetricApplicabilityRequests(
        status: MetricApplicabilityRequestStatus?,
        page: Int,
        size: Int,
        search: String?
    ): MetricApplicabilityRequestsResponse {
        validatePagination(page, size)

        val pageable = PageRequest.of(
            page,
            size,
            Sort.by(
                Sort.Order.desc("createdAt"),
                Sort.Order.desc("id")
            )
        )

        val specification = MetricApplicabilityRequestSpecification.getSpecification(status, search)
        val requestsPage = metricApplicabilityRequestRepository.findAll(specification, pageable)
        val currentUser = userInfoProvider.currentUser()

        val requests = requestsPage.content.map { request ->
            request.toResponse(currentUser)
        }

        val pendingCount = getVisibleRequestsCount(status)

        return MetricApplicabilityRequestsResponse(
            content = requests,
            page = requestsPage.number,
            size = requestsPage.size,
            totalElements = requestsPage.totalElements,
            totalPages = requestsPage.totalPages,
            pendingCount = pendingCount
        )
    }

    /**
     * Проверяет корректность параметров пагинации.
     */
    private fun validatePagination(page: Int, size: Int) {
        require(page >= 0) { "page не может быть меньше 0" }
        require(size in 1..MAX_PAGE_SIZE) { "size должен находиться в диапазоне от 1 до $MAX_PAGE_SIZE" }
    }

    /**
     * Возвращает количество видимых заявок выбранного статуса.
     *
     * Если статус не передан, возвращает количество всех заявок,
     * отображаемых Офису.
     */
    private fun getVisibleRequestsCount(status: MetricApplicabilityRequestStatus?): Long =
        if (status == null) {
            metricApplicabilityRequestRepository.countByIsVisibleInOfficeTrue()
        } else {
            metricApplicabilityRequestRepository.countByStatusAndIsVisibleInOfficeTrue(status)
        }

    /**
     * Преобразует заявку из БД в модель ответа API.
     */
    private fun MetricApplicabilityRequestEntity.toResponse(currentUser: UserDto): MetricApplicabilityRequestResponse {
        val assignment = requireNotNull(initiativeMetricAssignment) {
            "Для заявки id=$id отсутствует связь с initiative_metric_assignment"
        }

        val initiativeMetricType = requireNotNull(assignment.initiativeMetricType) {
            "Для assignment id=${assignment.id} отсутствует initiative_metric_type"
        }

        val initiative = requireNotNull(initiativeMetricType.aiAgent) {
            "Для initiativeMetricType id=${initiativeMetricType.id} отсутствует инициатива"
        }

        val metric = requireNotNull(assignment.metric) {
            "Для assignment id=${assignment.id} отсутствует метрика"
        }

        return MetricApplicabilityRequestResponse(
            requestId = requireNotNull(id),
            initiativeId = requireNotNull(initiative.id),
            initiativeName = initiative.agentName,
            metricId = requireNotNull(metric.id),
            metricName = metric.name,
            metricFrequency = metric.frequency,
            agentType = initiativeMetricType.agentType,
            requestStatus = status,
            applicabilityStatus = assignment.applicabilityStatus,
            comment = comment,
            resumePeriod = resumePeriod,
            requestedBy = getRequestedBy(createdBy, currentUser),
            requestedAt = requireNotNull(createdAt),
            availableActions = metricApplicabilityActionResolver.getAvailableActions(status, assignment.applicabilityStatus)
        )
    }

    /**
     * Возвращает отображаемое имя автора заявки.
     *
     * UserInfoProvider предоставляет только данные текущего пользователя,
     * поэтому его ФИО можно определить только для собственной заявки.
     * Для остальных пользователей временно возвращается их идентификатор.
     */
    private fun getRequestedBy(createdBy: Long, currentUser: UserDto): String {
        if (createdBy != currentUser.id) {
            return createdBy.toString()
        }

        return buildFullName(currentUser)
    }

    /**
     * Формирует ФИО пользователя из доступных частей имени.
     */
    private fun buildFullName(user: UserDto): String {
        val fullName = listOfNotNull(
            user.lastName?.trim()?.takeIf { it.isNotEmpty() },
            user.firstName?.trim()?.takeIf { it.isNotEmpty() },
            user.patronymic?.trim()?.takeIf { it.isNotEmpty() }
        ).joinToString(" ")

        return fullName.ifBlank { user.login ?: user.id.toString() }
    }
}

/**
 * Контроллер работы с заявками на неприменимость метрик.
 */
@Validated
@RestController
@RequestMapping("/api/v1/ai-agent")
class MetricApplicabilityRequestController(
    private val metricApplicabilityRequestService: MetricApplicabilityRequestService
) {

    /**
     * Возвращает очередь заявок на неприменимость метрик.
     *
     * Без status возвращаются заявки всех статусов.
     * Search выполняется по названию инициативы и названию метрики.
     */
    @GetMapping("/metric-applicability-requests")
    fun getMetricApplicabilityRequests(
        @RequestParam(required = false) status: MetricApplicabilityRequestStatus?,
        @RequestParam(defaultValue = "0") @Min(0) page: Int,
        @RequestParam(defaultValue = "20") @Min(1) @Max(100) size: Int,
        @RequestParam(required = false) search: String?
    ): MetricApplicabilityRequestsResponse =
        metricApplicabilityRequestService.getMetricApplicabilityRequests(status, page, size, search)
}


```
