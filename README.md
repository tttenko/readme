```java

/**
 * Заявка на неприменимость метрики для отображения в очереди Офиса.
 */
data class MetricApplicabilityRequestResponse(
    @field:Schema(description = "Идентификатор заявки", format = "int64")
    val requestId: Long,

    @field:Schema(description = "Идентификатор инициативы", format = "int64")
    val initiativeId: Long,

    @field:Schema(description = "Название инициативы", nullable = true)
    val initiativeName: String?,

    @field:Schema(description = "Идентификатор метрики", format = "uuid")
    val metricId: UUID,

    @field:Schema(description = "Название метрики", nullable = true)
    val metricName: String?,

    @field:Schema(description = "Периодичность сбора метрики", nullable = true)
    val metricFrequency: String?,

    @field:Schema(description = "Тип агента", example = "autonomous", nullable = true)
    val agentType: String?,

    @field:Schema(description = "Статус заявки", example = "PENDING")
    val requestStatus: MetricApplicabilityRequestStatus,

    @field:Schema(description = "Статус применимости метрики", example = "PENDING")
    val applicabilityStatus: MetricApplicabilityStatus,

    @field:Schema(description = "Комментарий к заявке")
    val comment: String,

    @field:Schema(description = "Период возобновления применимости метрики", format = "date", nullable = true)
    val resumePeriod: LocalDate?,

    @field:Schema(description = "Имя пользователя, создавшего заявку")
    val requestedBy: String,

    @field:Schema(description = "Дата создания заявки", format = "date")
    val requestedAt: LocalDate,

    @field:ArraySchema(arraySchema = Schema(description = "Действия, доступные пользователю для этой заявки"))
    val availableActions: List<MetricApplicabilityRequestAction>
)

/**
 * Страничный ответ со списком заявок на неприменимость метрик.
 */
data class MetricApplicabilityRequestsResponse(
    @field:ArraySchema(arraySchema = Schema(description = "Заявки на текущей странице"))
    val content: List<MetricApplicabilityRequestResponse>,

    @field:Schema(description = "Номер текущей страницы, начиная с нуля", format = "int32", example = "0")
    val page: Int,

    @field:Schema(description = "Размер страницы", format = "int32", example = "20")
    val size: Int,

    @field:Schema(description = "Общее количество заявок с учётом фильтров, без пагинации", format = "int64")
    val totalElements: Long,

    @field:Schema(description = "Общее количество страниц с учётом фильтров", format = "int32")
    val totalPages: Int,

    @field:Schema(
        description = "Общее количество заявок со статусом PENDING и isVisibleInOffice = true; не зависит от status, search и страницы",
        format = "int64"
    )
    val pendingCount: Long
)
```
